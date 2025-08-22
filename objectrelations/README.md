# Authorizing Object Relations in Kubernetes

Kubernetes comes with powerful mechanisms for authorizing actors like users, tools, applications or controllers to execute operations on resources on its data plane as well as for validating changes of resources done by those actors.
But despite all these mechanisms, it is not possible to enable or prevent the establishment of relationships between objects in a meaningful and controllable way. In the next sections we will see, what this concretely means, why it is required, what is possible today and how the problem could be solved.

## Object Relations

An important concept in Kubernetes is the possibility to establish relations between two objects. This is typically done by preparing the manifest structure of a resource type to include referential information to another object. With its resource type (kind and api group), a namespace and an object name, every object in a Kubernetes dataplane has a unique identity over space (not time). The same name could be reused later in time to create a new object, after the old one has been deleted.  Additionally, it features UID, which is also unique over time.
Typically, the named identity is used to describe such referential fields. A typical example is an object representing a digital twin for a real world object requiring some credential to be used.

> In Kubernetes itself, the most prominent example is using a `Secret`, for example by a `Pod`, which uses container images loaded from an image repository. This might require credentials. Therefore, the Pod has a specification field `imagePullSecrets`.

Such a referential field might have several manifestations:
- it must at least describe the name of the used object. This is possible if the resource type is implied by the meaning of the field (for example the image pull secret reference is always a reference a `Secret` resource.)
- if objects of different types should be addressable, the type information must be included
- if the object is namespaced and cross namespace references should be possible, a namespace field must be included.
- If uniqueness over time is required an object UID can be used.

Typically, the UID is not used, because this would require to adapt all using objects, if accidentally the referenced resource has been deleted and recreates.

> Kubernetes provides with `k8s.io/api/core/v1.ObjectReference` even a standard type for the layout of a resource specification.

So, what we can see, an explicit object relation in the Kubernetes Resource Model (KRM) is typically established by a field in the using resource. It is a tupe *(s,r,t)*, wher *s* is the source object, holding the reference, *r* is the *relation type*, the meaning or *purpose* the referenced object should be used for by *s*, and *t* is the *target* object of the relation, the referenced object.

More complex n-ary relations are not used, although an object featuring multiple binary relations could be seen as an n-ary relation *(s,r,t<sub>1</sub>,...t<sub>n</sub>)*, where *r* is the resource type. So, it is sufficient to deal with binary relations.

### Why restricting object relations.

A controller responsible for a resource type uses the given referenced object to implement the object in the real work (or its target environment) using the content of the referenced resource or its real world instance, It does not know, and it cannot know, whether this should be allowed or not. Therefore, there must be a way to prevent establish a relation, which should not be possible.

For example, a secret describes the access to a technical environment, which should not be usable by all objects of a resource type describing a relation to such an environment.

### Isn't it a regular authorization problem
The first impression is, that this could be the task of the
authorization environment used to authorize modification (or creation) operations on objects. Only selected actors (principals in authorization systems or subjects in Kubernetes) should be able to establish particular relations by setting the reference field accordingly. This assumption may be true within a trust domain, but not when objects from multiple trust domains are involved. An object should usable from a particular sets of other namespaces, but not accessible for users in those namespaces, regardless whether they have admin privileges or not.

In Kubernetes a namespace is a trust-domain for authorizations on namespaced resources. Let's assume we are mainly talking about such resources. One idea of such a trust domain in Kubernetes is, that authorizations can be managed by domain-local administrators. This is fine, as long as both involved object live in the same namespace. The admin could explicitly grant permissions for establishing particular relations for selected users, or no user,if the relation should generally not be possible. The only requirement is to have such a fine-grained access control. With *Validating Admission Webhooks* this would basically be possible even today, but not bóu-of-the-box.
[Later](#operation-authorization-and-resource-validation-in-kuberbetes) we will see what is possible and what not. But our problem is not limited to objects in a single namespace.

If we try to solve the general problem with authorizations in a single trust domain, the domain admin would be able to enable the usage of any object of any other namespace. But this is definitely not intended. We need a separation of responsibility.
The trust domain of the potentially usable object must be the authorization domain for granting the possibility to use objects in this domain by other domains (here namespaces). And the admin of the using domain should be responsibility do enable particular actors to establish such a relation, but only if it hb been granted by the providing domain.

So, what can be [observed](#separation-of-operation-authorization-and-value-validation) is, that the release of an object to be referencable is not an operational authorization problem, it is more a value validation problem. And in both involved domains the operation authorization handles who can release on object to be usable, or who can finally configure such a relation for a particular object.

One may be tempted to solve the problem by using permissions to grant appropriate permissions required to establish such relations. But this will lead into a dead end, because:
- the resulting authorizations will become incredibly complicated.
- they are cross-domain, which means that they cannot be managed inside a domain, but by a system administrator.

And finally, it would not express the actual meaning.

Because of these problems often cross-namespace references are not supported by the resource type (or by the controller). If urgently required for a particular kind of resource a typical solution is to introduce proxy resources working as indirection.
For example a `SecretRef`resource whose sole task is to reference 
a secret in another namespace. With RBAC the creation or modification is prohibited by namespace users and admins. If a secret should be reusable the cluster admin has to create such an object in the intended target namespace. But the price is not only to require the cluster admin to enable the usage by creating such a proxy object, but the controller must be prepared for this, also. They have to support references to the
direct object type (for example secrets), and the proxy resource type, also. This might be a solution for some special cases, but is not a practical solution for the general use case.


## Operation Authorization and Resource Validation in Kubernetes

Let's first have a look at the existing mechanism in Kubernetes
and see what can be done with it.

### RBAC

First of all, there is RBAC, which is an acronym for Role-Based-Access-Control. It is solely meant for authorizing operations on objects stored in the Kubernetes data plane or for object-less operations in a cluster. Principals, called *subjects* in Kubernetes, may be users, service accounts as representatives for machine users applications or controllers, or any actor whose identity can be expressed as string. It is possible grant permissions for data plane operations like get, list, watch and put on resources, resource types and or namespaces. Operation targets are described by role objects, which can be bound to subjects by binding objects. To enable namespace local authorization management, there are cluster wide versions of those resources and namespace-local ones. A cluster admin can grant permissions for managing namespace-local roles and bindings to specific actors by cluster-wide versions.

This mechanism does not support permissions depending on features of an object, like annotations, labels or fields. There are projects dealing with extensions enabling such possibilities.

Those extensions would enable to restrict the usage of particular references for modifications of an object.

### Validating Admission Webhooks.

Once a modification request is authorized validating webhooks are used check whether a requested modification of resource (including creation and deletion) is valid. Therefore, the hook provides the old manifest version together with the requested new one, and the hook can accept or reject the requested change.
It can therefore be used to check for valid field values or the consistency of correlated fields as well as allowed changes (for example for read-only attributes).

The focus here lies on the validation of the given resource manifest. This is important for non-standard resource types (whose validation is done by the API server), which can be declared by CRDs (Custom Resource Definitions). Providing such a webhook can be added to provide type specific checks. As with Kubernetes v1.25, many such checks can be already described by CEL expressions as part of the OpenAPI Schema specification.

> Kubernetes added `x-kubernetes-validations` as an extension to OpenAPI v3.
> ```yaml
>    kind: CustomResourceDefinition
>    metadata:
>      name: widgets.example.com
>    spec:
>      group: example.com
>      names:
>        kind: Widget
>        plural: widgets
>      scope: Namespaced
>      versions:
>      - name: v1
>        served: true
>        storage: true
>        schema:
>          openAPIV3Schema:
>            type: object
>            properties:
>              spec:
>                type: object
>                properties:
>                  size:
>                    type: integer
>                  maxSize:
>                    type: integer
>            x-kubernetes-validations:
>            - rule: "self.size <= self.maxSize"
>              message: "spec.size must not exceed spec.maxSize"
>  ```

But the admission API also provides information about the requesting subject. THis way it can not only be used for manifest validation, but it can also reject modifications not allowed for the requesting user. Therefore, validating admission webhooks can de-facto also be used to implement operation authorizations. Because they are only called for modifications, read-authorizations cannot be handled.

This kind of hybrid-functionality is therefore not sufficient to really implement an authorization system based on attribute based conditions, it is basically only valid for value validation and it [mixes validation with authorizations](#separation-of-operation-authorization-and-value-validation).

Another possibility to gain full control over such an extensive validation is to implement an API group by an extension api server (or aggregated API server) instead of a set of regular controllers.

### Validation Admission Policies

As of version v1.30 Kubernetes supports Validation Admission Policies. They were introduced to replace the need for coding Validation Admission Webhooks with a standardized declaration-based approach controlled via regular resources.

It is based on two new resources `ValidatingAdmissionPolicy` and ValidatingAdmissionPolicyBinding`, both are cluster-scoped.
A policy desclares validation rules on some resource types.

> ```yaml
> apiVersion: admissionregistration.k8s.io/v1
> kind: ValidatingAdmissionPolicy
> metadata:
>   name: require-team-label
> spec:
>   matchConstraints:
>     resourceRules:
>       - apiGroups: [""]
>         apiVersions: ["v1"]
>         operations: ["CREATE","UPDATE"]
>         resources: ["pods"]
>   validations:
>     - expression: "has(object.metadata.labels['team'])"
>       message: "All Pods must have a 'team' label"
> ```

And a binding binds a policy against some objects, namespaces and/or subjects.

> ```yaml
> apiVersion: admissionregistration.k8s.io/v1
> kind: ValidatingAdmissionPolicyBinding
> metadata:
>   name: require-team-label-binding
> spec:
>   policyName: require-team-label   # The policy to bind
>   # Bind only to certain users or service accounts
>   subjects:
>     - kind: User
>       name: "alice@example.com"
>     - kind: Group
>       name: "dev-team"
>     - kind: ServiceAccount
>       name: "ci-pipeline"
>       namespace: "ci"
>   # Apply only in specific namespaces
>   namespaceSelector:
>     matchExpressions:
>       - key: environment
>         operator: In
>         values: ["prod", "staging"]
> ```

<a id="vap-excursus"><a/>

Additionally, a parameterization is supported. A policy can declare and parameters, which will be bound by bindings. Hereby, the value may be taken from a kind of resource, found in a specific namespace or in the namespace of the for which the policy is checked. This can be used to reuse a policy in a context specific way with context specific values. The context thereby is given by the combination of binding and parameter resource, which might again be namespace specific (if no namespace is given). With this, it is possible to influence the policy inside a namespace ny namespace-local resources.

At a first glance this looks like a possible solution for our cross-namespace usage scenario. It could look like this. A policy checks whether the secret name is in a list of allowed names.

> ```yaml
> apiVersion: admissionregistration.k8s.io/v1alpha1
> kind: ValidatingAdmissionPolicy
> metadata:
>   name: restrict-secret-usage
> spec:
>   failurePolicy: Fail
>   parameters:
>     - name: allowedSecrets
>       type: string[]
>   validations:
>     - expression: "self.spec.secretRef.name in parameters.allowedSecrets || !self.spec.secretRef.namespace == 'namespace1'"
>       message: "This resource can only reference approved secrets in namespace1"
> ```

And the binding binds the policy to a concrete validation resource containing the allowed names.
> ```yaml
> apiVersion: admissionregistration.k8s.io/v1alpha1
> kind: ValidatingAdmissionPolicyBinding
> metadata:
>   name: bind-secret-usage
> spec:
>   policyName: restrict-secret-usage
>   parameterResource:
>     apiVersion: v1
>     kind: ConfigMap
>     name: allowed-secrets
>   parameters:
>     allowedSecrets:
>       valueExpression: "parameterResource.data['allowedSecrets'].parse_json()"
>   matchResources:
>     resourceRules:
>       - apiGroups: [""]       # adjust for your resource type
>         apiVersions: ["mygroup/v1"]
>         resources: ["mykind"]   # or the relevant resource type
> ```

Finally, the namespace provides a parameter resource (here a `ConfigMap`):
> ```yaml
> apiVersion: v1
> kind: ConfigMap
> metadata:
>   name: allowed-secrets
>   namespace: namespace2
> data:
>   allowedSecrets: '["secret-a","secret-b"]'
> ```

This way a namespace can enable the usage of secrets from other namespaces by a config map in its own namespace.
But this is not the requirement. Fortunately the namespace of a
parameter resource can be fixed to the providing namespace (namespace1).
It then can use a local object to release, for example, a secret to other namespaces by listing the grants.

The policy now checks whetjer the requested
> ```yaml
> apiVersion: admissionregistration.k8s.io/v1alpha1
> kind: ValidatingAdmissionPolicy
> metadata:
>  name: restrict-secret-usage
> spec:
>  validationFailureAction: Enforce
>  matchConstraints:
>   # Apply to all resources that might reference secrets
>   resourceRules:
>    - apiGroups: ["*"]
>      apiVersions: ["*"]
>      resources: ["*"]
>  parameters:
>   - name: secretNamespace
>     schema:
>      type: string
>   - name: visibility
>     schema:
>      type: object
>  # CEL expression
>  CEL:
>   expressions:
>    - expression: |
>       # Only apply if the object has a secretRef field
>       !has(self.spec.secretRef) ||
>       # Secret must come from a different namespace → allow
>       self.spec.secretRef.namespace != parameters.secretNamespace || (
>           # Secret must exist in the visibility map
>           has(parameters.visibility[self.spec.secretRef.name]) &&
>           # Target namespace must be listed under that secret
>           has(parameters.visibility[self.spec.secretRef.name][object.metadata.namespace]) &&
>           # Within that namespace’s list, there must be a matching {kind, name}
>           exists(p, parameters.visibility[self.spec.secretRef.name][object.metadata.namespace],
>             p.kind == object.kind && p.name == object.metadata.name
>           )
>       )
> ```

> ```yaml
> apiVersion: admissionregistration.k8s.io/v1alpha1
> kind: ValidatingAdmissionPolicyBinding
> metadata:
>   name: bind-secret-usage
> spec:
>   policyName: restrict-secret-usage
>   parameterResource:
>     apiVersion: v1
>     kind: ConfigMap
>     name: secret-visibility
>     namespace: namespace1   # fixed here
>   parameters:
>     visibility:
>       valueExpression: "parameterResource.data['visibility'].parse_json()"
>     secretNamespace:
>       value: "namespace1"
> ```

> ```yaml
> apiVersion: v1
> kind: ConfigMap
> metadata:
>   name: secret-visibility
>   namespace: namespace1
> data:
>   # JSON structure: secret -> namespace -> [{kind, name}]
>   visibility: |
>     {
>       "secret-a": {
>         "namespace2": [
>           {"kind": "Pod", "name": "mypod"},
>           {"kind": "Deployment", "name": "web"}
>         ],
>         "namespace3": [
>           {"kind": "Deployment", "name": "api"}
>         ]
>       },
>       "secret-b": {
>         "namespace2": [
>           {"kind": "StatefulSet", "name": "db"}
>         ]
>       }
>     }
> ```

This looks great, the providing namespace can explicitly release a secret for dedicated use cases. Maybe, with even more complex rules, taking a value `*` into account.

But unfortunately it has some fundamental drawbacks:
- The binding must contain a constant namespace to take the parameter resource from and to pass to the policy to bind together a parameter resource and secret ref namespace the policy should be used for.
 Basically this information is known by the object checked (the namespace of the referenced secret), but a binding neither has access to the checked object nor is there a way back from the policy to the binding.
  Therefore, every providing namespace requires a new binding, which cannot be established by the namespace.
- Because the structure of different resources containing a secret reference is typically different, a separate policy is required for every consuming resource type or .
- The release information must be in single parameter resource, because there is match if the policy matches at lease one parameter resource, it must match always all parameter resources.

It should be obvious, that this approach ends up in a proliferation of complex policies and bindings, which all must be declared centrally, and do not directly express the original intend anymore.


## Separation of Operation Authorization and Value Validation.

Instead of trying to reuse, extend or adapt existing technical solutions, we should carefully examine the problem space and clearly distinguish between different meanings.

Only because a problem can be (partly) solved by a similar solution implementation, this does not necessarily mean, that the same description methodology should be offered for a user.
It could require different or  different set of attributes to describe a request, or, even more important, different responsibilities could be involved.

On the other side this also does not necessarily mean, that behind the scenes a common implementation may not be used, called
by different frontends.

So, let's have a closer look at the problem space covered by the today's mechanisms in Kubernetes.

First of all, there are obviously two different tasks:
- *Value Validation*: validate whether fields and field values of an object are valid
- *Authorization Validation*: validate whether a requested operation if allowed for the requesting principal.

It is something completely different, whether a particular user is allowed to set a dedicated value combination for a field or whether this combination is possible at all.

### Value Validation

The sole task of value validation is to avoid the setting of field/value combinations of an object which is not valid or possible.

But having a closer look this is not a single task. Whether a field/value combination may depend on two different kinds of constraints arising from two different responsibilities:
- *Domain Validation*: The resource type has a particular meaning determined. And this meaning implies constraints for value and field combinations. For example a component of a RGB color value is always a number in the range 0<=x<=255. This has nothing to do with the context the color should be used, it is completely defined by the meaning behind the element (although it might technically be an integer). Or those constraints might be given by the underlying elements of the real world finally represented by the resource object.
- *Context Validation*: A particular resource object might be used in a concrete context. This context is typically identified by some label, annotation of field setting defined by an application or group of users in a namespace. There may be any number of such contexts per cluster. The task here is to be able to limit field/value combinations according to requirements of this usage context. Those limitations are not intrinsic to the resource type.

As we can see both kinds of limitations are subject to different areas of responsibility. While the constraints for the domain validation are arising from the resource type itself and are the same and enforced for every object for every application of this resource type in a cluster, the context constraints are arising from the usage of particular resources in a particular cluster.
So, responsible for the first flavor is the provider of a resource type, and for the seconds flavor some principals in a particular cluster. In the first case, all objects are (and must be) affected, in the second only objects that belong to a specific context.

This differentiation must be reflected in the API used to declare such validations.
This first one is usefully part of the schema definition of the resource type wherever possible. More dynamic checks, for example checking value transitions, can be done by more expressive elements. In this are Validating Admission Webkooks and Validating Admission Policies can be used. They are valid cluster-wide. But they should no check context constraints.

The second kind of constraints must be declarable by principals
responsible for particular usage contexts, typically inside a namespace. And here we easily can see, that Validating Admission Policies cannot be used, because they are cluster-scoped resources. And this although, concerning their expressiveness, they would be able to describe such constraints. So, the design of this new mechanism is not applicable for a common use case according the semantics of the problem domain, it is just an adaptation of the already existing mechanism of Validating Admission Webhooks. To use it to solve the context problem, you have to leave your trust-domain and require cluster privileges.

For sure, those two kinds of constraints have nothing to do with 
the wish to restrict users in a context or trust domain to set specific values or value combinations. For example, only selected users should be able to set a high priority. All those values are valid in the context, not all users may set those values.

A basic takeaway is, that it would be a good practice to use some *Policy* resources only for context-related constraints. And handling of such resources must be authorizable locally to the related namespace. Thi expresses the local character of such constraints in contrast to constraints intrinsic to a particular resource type.

Returning to our initial problem with cross-namespace references we can see the following:
- the domain validation should accept such references, because there is no technical constraint against it. 
- it is a matter of context validation. But it cannot completely be solved inside a namespace, because one element is missing. The referenced object belongs to different trust domain we need something that authorizes a context to use it. By the way, this could even be useful inside a single namespace, if it hosts multiple usage contexts. But again, the usage of an object must still be released by the owning side, not the consuming one. So even inside a namespace with regular authorization mechanisms, a principal is required which has cross context privileges.

### Authorization

In the last sections we have seen, that value validation is formally a different task than authorization validation.
While value validation is responsible to handle the question,
*what* is valid in a particular context, authorization is responsible to answer the question, *who* may use it.

An authorization system for a multi-trust-domain environment must at least meet some basic requirements:
- a principal may never grant permissions he does not have to other users.
- there must be cross domain permissions which enabled a cross-domain admin to grant permissions to a principal to play the role of an admin for a single trust domain.
- Trust-domain admin permissions meant that a principal with those permissions can control the access to objects in his trust-domain.
- a trust-domain admin should never need to know users having permissions of other trust domains nor manage permissions of other trust-domains (despite this has been granted explicit permissions to do so (multi-domain admins))
- a trust-domain admin (even not being a multi-domain admin) must be able to control the potential usage of objects in his trust-domain by other trust domain.

Especially the last point is important for our usage scenario.
These basic requirements mut be reflected in design of the description layer for managing permissions, because their maintenance must be authorized like the maintenance for all the other objects.

What should be describable for authorizations and how should those descriptions be organized? Let's have a look at the existing mechanisms.

RBAC is able to answer the question, who is able to execute a particular operation on an object. Here, only the operation type (like GET,PUT,...) and the identity of the concerned object is taken into account. The RBAC system uses `Role`s to describe permissions for objects inside a namespace and appropriate `RoleBinding`s to assign those roles to subjects. Both kinds of resources live in a namespace.  `ClusterRole`s and `ClusterRoleBinding`s are cluster-scoped objects and used to maintain cluster wide permissions
on cluster-scoped or namespaced objects. This structure allows to asign namespace-only permissions to dedicated subjects and therefore meets the above requirements (point 1 is not considered here in detail).

To maintain authorizations in finer granularity than referring to namespaces and objects within is not really possible.

With validation admission webhooks only modifying requests can be handled, but no read, watch or list requests. Basically they should only be used to handle value validation, but can technically used for implementing write authorization models. They don't have a description layer, they are just specific implementations.

Validating Admission Policies are a dedicated implemenation of a an validation admission webhook and therefore underly the same constraints. Their task is offer a formal declarative configuration with resource objects to describe again, value chackes as weel as write authorizations.  They allow to describe modification authorizations based on labels, annotations and fields and their values of modified objects. But there are several more severe constraints:
- it is not possible to follow relations between objects. For example: you may establish only a reference to a secret with a particular label value.
- in contrast to RBAC roles and bindings, such policies cannot be maintained under the control of a namespace, their maintenance always requires a cluster admin. All description types are cluster-scoped.

This basically fine for value validation, but only for domain validation. Context validation is technically possible, but requires cluster-scoped permissions. As the [excursus above](#vap-excursus) has shown using parameterization allows to delegate policy argument management to namespaces, but t always requires a dedicated preparation on the cross-namespace level. Therefore, the minimal requirements for a description layer are not fulfilled.

Because of those limitations there are projects trying to
introduce a generic authorization engine like Cedar.
But POCs in this area are always limited by the constraints of admission web hooks, which are typically used to hook into the authorization process. And as we have seen, this mechanism
is not generally suitable for solving authorization problems. 

But even under the assumption that attribute-based conditional authorizations (Cedar does support even to follow references) would be available, would it be possible to resolve our referencing object problem?

Here, we have to see that regular authorizations always authorize principal to do something, for example setting a reference field to a dedicated value. But as we have seen in previous section,
the problem is primarily not related to the question, *who* may establish a reference, but is such a reference allowed, it's some kind of context validation.

### Solving object relation constraints with traditional  Authorization

But even if we would be content with such an authorization solution, we still have the problem with the responsibility. Authorizations
of such kind are either maintained in a namespace for objects in this namespace or by cluster wide settings, which require some cluster-wide permissions. This is definitely not what the problem domain requires.

So, some kind of authorization seems to be missing. Indeed, there are two kinds of authorizations:
- *operational authorizations* control who may execute which operation on a target object.
- *relational authorizations* control, which relation between objects is allowed in the context of a data plane.

The second kind of authorizations could be seen as authorizations where the principal role is fed by an object and the operation is the kind of relation. But this is an in implementation point of view. As usual, the problem arises with the responsibilities.

Let's assume we just extend RBAC to allow principal identities describing an object identity and the operation to describe arbitrary relation names. This does not change anything for the
responsibilities for the maintenance of Role and bindings. They are either cluster-wide or namespace-local. This means inside a namespace only roles are observed hosted by this namespace, or better the other way around, roles and bindings may refer only to namespace-local objects. Even if the is weekend and bindings (or even worse roles) could refer to objects from other namespaces, the providing namespace would lose control over its objects.
And this is exactly what should be avoided in the required scenario. The release of objects to be usable in another namespace must be under the control of the providing namespace, not of the consuming namespace, and the control over the subjects wha may establish such relations must be under the control of the consuming namespace,

So, we need something, where the providing context must describe authorization resources authorizing foreign contexts to be able to use a local object, the target side of the relation, and the target context must describe in its namespace which using object might be affected and then who may establish such a relation.
THis means, sticking to the operational Role and Binding concept, the binding now must describe the (target) object part and the role the consuming (principal) part, their terms subject and resource do not match anymore, because subject should somehow denote who initiates something. And for object relations, this is the using object, because it describes what object it wants to use (not the other way around).

Therefore, independently of the chosen underlying implementation,
another declaration layer has to be introduced, which meets the requirements of object relational authorizations.

## Possible solutions

### Preconditions

As we have seen, using the traditional authorization system, even with the possibility for attribute based conditions are not suitable for describing constraints for object relations.
Because of this the introduction of conditions is not a precondition for the support of relational object authorizations. Those tracks can be followed separately. Once there is an engine able to handle this,
it could very likely also be used to implement the basic authorization checks for object relations (using objects also as principal).
But this is not required, neither for the description layer nor for the way those checks have to be executed. The description layer can ba extended to offer appropriately extended policies and the check execution does not (need to) know anything about the policy declaration.

But there is another crucial point. If relations should be validated, it must be possible to identify those relations at the place the are defined, the resource manifest. If we don't want to fall back to the need of describing values of fields with simple data types, but use a more logical relation view in our description layer, the API model must be able to describe, what fields hold a reference, how it is specified and what is its purpose (the relation kind). Only this way it is possible for the API server to map the logical view, object X uses object Y with purpose R, to field values in a resource manifest.

### Declaration Layer

To get started easily we can use the layout of the RBAC resources. But we to reflect the fact that the roles of subject and resource are reversed.

So, instead of `ClusterRole` and `Role`, we have a  `ClusterObjectRole`and `ObjectRole`. But they now describe the referencing objects (formally the principal) and the relation type or purpose of the required relation.

For example:
> ```yaml
> apiVersion: relationalaccesscontrol.k8s.io/v1alpha1
> kind: ObjectRole
> metadata:
>   name: ingres-dns-access
>   namespace: application
> rules:
> - apiGroups:
>   - ""
>   resources:
>   - ingress
>   resourceNames:
>   - *
>   relations:
>   - use
> ```

They are managed in namespace of the consuming object.
They describe, which of the local objects should be able to consume
some resource. This decision is completely handled in the local trust-domain.

On the other side there are the binding. `ClusterObjectRuleBinding` and `ObjectRoleBinding`.  Those resources are maintained on the side of the provided object and are used to enable the usage of some local object.

For example:

```yaml
apiVersion: relationalaccesscontrol/v1alpha1
kind: ObjectRoleBinding
metadata:
  name: ingres-dns-access
  namespace: kube-system
targets:
- apiGroups:
  - loadbalancer.gardener.cloud
  resources:
  - dnsprovider
  resourceNames:
  - clusterdns
roleRef:
  apiGroup: relationalaccesscontrol.k8s.io
  kind: ObjectRole
  namespace: kube-system
  name: ingres-dns-access
```

The usagge context the access is granted is herby represented by the referenced role. For ObjectRole those references must always be namespaces, because they have to be able to grant access to contexts outside the own namespace.


This is very basic model similar to RBAC, let's call it OBAC (Object Based Access Control).

If later a new general authorization engine is used, it can also be fed by those rules. If conditions are possible the extension can be done similar to the extension done to RBAC (or an appropriate replacement).

We should avoid to be too fast with OBAC and introduce features not available to operational access control, to be able to do such an extension in a uniform way for both, operational and relational access control.

### Permission Enforcement

The declaration layer part is the easiest part of some relational authorization system. The much harder part is the enforcement of the described permissions.





