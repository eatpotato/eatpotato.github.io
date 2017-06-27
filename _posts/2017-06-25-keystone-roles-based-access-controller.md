---
layout:     post
title:      openstack各组件的RBAC(Role Based Access Controll)
date:       2017-06-25
author:     xue
catalog:    true
tags:
    - openstack
---

以kilo版本的openstack为例

## Keystone

#### Role 和 Policy 策略

验证用户username和password以后，用户有没有权限来执行某个操作则取决于 Role 和 基于 Roles 的鉴权。

Keystone API V3 与 V2 相比，对 policy 的支持有很多的增强。V2 的支持和 OpenStack 其它组件比如cinder，nova 等差不多，只支持基于 policy.json 文件的 policy 控制；而 V3 中，支持对 policy 的 CURD 操作，以及对 endpoint，service 等的 policy 操作等。

这里主要介绍一下V2版本的RBAC

Keystone 在每个子模块的 controller.py 文件中的各个控制类的方法上实施 RBAC。有三种方法

1.在函数体中使用 self.assert_admin(context)  
2.使用装饰器@controller.protected()
3.使用装饰器@controller.filterprotected('type', 'name')

来看一下他们各自的实现

#### assert_admin

```
keystone.common.wsgi.Application.assert_admin:
    # 如果context中is_admin是True， 直接跳过下面的验证
    if not context['is_admin']:
        try:
            user_token_ref = token_model.KeystoneToken(
                token_id=context['token_id'],
                token_data=self.token_provider_api.validate_token(
                    context['token_id']))
        except exception.TokenNotFound as e:
            raise exception.Unauthorized(e)

        validate_token_bind(context, user_token_ref)
        creds = copy.deepcopy(user_token_ref.metadata)

        try:
            creds['user_id'] = user_token_ref.user_id
        except exception.UnexpectedError:
            LOG.debug('Invalid user')
            raise exception.Unauthorized()

        if user_token_ref.project_scoped:
            creds['tenant_id'] = user_token_ref.project_id
        else:
            LOG.debug('Invalid tenant')
            raise exception.Unauthorized()

        creds['roles'] = user_token_ref.role_names
        # Accept either is_admin or the admin role
        self.policy_api.enforce(creds, 'admin_required', {})
    ↓
keystone.policy.backends.rules.Policy.enforce:
    enforce(credentials, action, target)
    ↓
keystone.policy.backends.rules.enforce:
    return _ENFORCER.enforce(action, target, credentials, **extra)
    ↓
oslo_policy.policy.Enforcer.enforce(self, rule, target, creds, do_raise=False,
                exc=None, *args, **kwargs):
    # 这里的rule为admin_required，target为{},creds为包含user_id,tenant_id,tenant_id的字典
    self.load_rules()
    
    # Allow the rule to be a Check tree
    # 这库的
    if isinstance(rule, _checks.BaseCheck):
        result = rule(target, creds, self)
    elif not self.rules:
        # No rules to reference means we're going to fail closed
        result = False
    else:
        try:
            # Evaluate the rule
            result = self.rules[rule](target, creds, self)
        except KeyError:
            LOG.debug('Rule [%s] does not exist' % rule)
            # If the rule doesn't exist, fail closed
            result = False

    # If it is False, raise the exception if requested
    if do_raise and not result:
        if exc:
            raise exc(*args, **kwargs)

        raise PolicyNotAuthorized(rule, target, creds)

    return results   
```

下面这行代码比较关键，rule的值为'admin_required，根据/etc/keystone/policy.json文件读出来的值为(role:admin or is_admin:1),所以selef.rules[rule]的type是
\<class 'oslo_policy._checks.OrCheck'\>

```
result = self.rules[rule](target, creds, self)
```

再来看看OrCheck的代码:

```
class OrCheck(BaseCheck):
    def __init__(self, rules):
        self.rules = rules
        
    def __call__(self, target, cred, enforcer):
        """Check the policy.

        Requires that at least one rule accept in order to return True.
        """

        for rule in self.rules:
            if rule(target, cred, enforcer):
                return True
        return False

```


#### @controller.protected()

```
keystone.common.controller.protected:
    def wrapper(f):
        @functools.wraps(f)
        def inner(self, context, *args, **kwargs):
            if 'is_admin' in context and context['is_admin']:
                LOG.warning(_LW('RBAC: Bypassing authorization'))
            elif callback is not None:
                prep_info = {'f_name': f.__name__,
                             'input_attr': kwargs}
                callback(self, context, prep_info, *args, **kwargs)
            else:
                action = 'identity:%s' % f.__name__
                creds = _build_policy_check_credentials(self, action,
                                                        context, kwargs)

                policy_dict = {}
                if (hasattr(self, 'get_member_from_driver') and
                        self.get_member_from_driver is not None):
                    key = '%s_id' % self.member_name
                    if key in kwargs:
                        ref = self.get_member_from_driver(kwargs[key])
                        policy_dict['target'] = {self.member_name: ref}

                
                if context.get('subject_token_id') is not None:
                     token_ref = token_model.KeystoneToken(
                        token_id=context['subject_token_id'],
                        token_data=self.token_provider_api.validate_token(
                            context['subject_token_id']))
                    policy_dict.setdefault('target', {})
                    policy_dict['target'].setdefault(self.member_name, {})
                    policy_dict['target'][self.member_name]['user_id'] = (
                        token_ref.user_id)
                    try:
                        user_domain_id = token_ref.user_domain_id
                    except exception.UnexpectedError:
                        user_domain_id = None
                    if user_domain_id:
                        policy_dict['target'][self.member_name].setdefault(
                            'user', {})
                        policy_dict['target'][self.member_name][
                            'user'].setdefault('domain', {})
                        policy_dict['target'][self.member_name]['user'][
                            'domain']['id'] = (
                                user_domain_id)

                policy_dict.update(kwargs)
                self.policy_api.enforce(creds,
                                        action,
                                        utils.flatten_dict(policy_dict))
                LOG.debug('RBAC: Authorization granted')
            return f(self, context, *args, **kwargs)
        return inner
    return wrapper
```

通过这种方法最终也是调用了self.policy_api.enforce来鉴权，不过注意到其中的这行代码：

```
action = 'identity:%s' % f.__name__
```

所以,在控制类的方法上加上此装饰器，如：

```
@controller.protected()
    def create_domain(self, context, domain):
```

对面的action即为identity:create_domain

#### @controller.filterprotected('type', 'name')

```
@controller.filterprotected('enabled', 'name')
    def list_domains(self, context, filters):
```

具体的实现代码和protected类似。

## nova


以创建虚拟机为例

```
nova.compute.api.API.create:
    self._check_create_policies(context, availability_zone,
                                requested_networks, block_device_mapping)
    ↓
nova.compute.api.API._check_create_policies:
        """Check policies for create()."""
        target = {'project_id': context.project_id,
                  'user_id': context.user_id,
                  'availability_zone': availability_zone}
        if not self.skip_policy_check:
            # 检查创建虚拟机的权限
            check_policy(context, 'create', target)

            if requested_networks and len(requested_networks):
                check_policy(context, 'create:attach_network', target)

            if block_device_mapping:
                check_policy(context, 'create:attach_volume', target)
    ↓
def check_policy(context, action, target, scope='compute'):
    _action = '%s:%s' % (scope, action)
    nova.policy.enforce(context, _action, target)
    ↓
nova.policy.enforce:
    init()
    credentials = context.to_dict()
    if not exc:
        exc = exception.PolicyNotAuthorized
    try:
        result = _ENFORCER.enforce(action, target, credentials,
                                   do_raise=do_raise, exc=exc, action=action)
    except Exception:
        credentials.pop('auth_token', None)
        with excutils.save_and_reraise_exception():
            LOG.debug('Policy check for %(action)s failed with credentials '
                      '%(credentials)s',
                      {'action': action, 'credentials': credentials})
    return result
    ↓
nova.openstack.common.policy.Enforcer.enforce:
    self.load_rules() #重新读取 policy.json 文件
    ...
    result = self.rules[rule](target, creds, self) #使用特定的 rule 来检查 user 的权限
    ...
    return True or PolicyNotAuthorized(rule)

```


## cinder

以创建cinder volume为例

```
cinder.volume.flows.api.create_volume.ExtractVolumeRequestTask.execute:
    # ACTION = 'volume:create'
    policy.enforce_action(context, ACTION)
    ↓
cinder.policy.enforce_action:
    return enforce(context, action, {'project_id': context.project_id,
                                     'user_id': context.user_id})
    ↓
cinder.policy.enforce:
    init()

    return _ENFORCER.enforce(action, target, context.to_dict(),
                             do_raise=True,
                             exc=exception.PolicyNotAuthorized,
                             action=action)
    ↓
cinder.openstack.common.policy.Enforcer.enforce:
    #rule:volume:create
    #target: {'project_id': u'xx', 'user_id': u'xx'},  
    #creds 中包括 user role 等信息
    self.load_rules() #重新读取 policy.json 文件
    ...
    result = self.rules[rule](target, creds, self) #使用特定的 rule 来检查 user 的权限
    ...
    return True or PolicyNotAuthorized(rule)

```

## 参考
[世民谈云计算](http://www.cnblogs.com/sammyliu/p/4277178.html)
