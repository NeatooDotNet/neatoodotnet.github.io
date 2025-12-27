# TODO: Documentation Clarifications Needed

These items require clarification to ensure the documentation accurately reflects the Neatoo and RemoteFactory frameworks.

---

## TODO 1: AuthorizeFactoryOperation Enum Values

Are these enum values correct?

| Value | Bit |
|-------|-----|
| Create | 1 |
| Fetch | 2 |
| Insert | 4 |
| Update | 8 |
| Delete | 16 |
| Read | 32 |
| Write | 64 |
| Execute | 128 |

**Answer:**

No, Read is 32, write is 128 and execute is 128
They are a bitwise. They are used to define the enum FactoryOperation.

## TODO 2: ValidateBase with RemoteFactory

Is validation strictly a Neatoo feature, or can RemoteFactory users access ValidateBase/validation capabilities independently?

**Answer:**

Validation is strictly a Neatoo feature.

## TODO 3: Registration Method for RemoteFactory-Only

Should there be separate `AddRemoteFactoryServices()` documentation for RemoteFactory-only usage, distinct from `AddNeatooServices()`?

**Answer:**

We want this site to be a Neatoo documentation site. The fact that RemoteFactory is seperable should be documented in it's own page but everywhere else it should be assumed you are using full Neatoo. So, when talking about Value Objects and Authorization just referr to it as 'Neatoo'. 

Note: This is another change in thining going forward. In this site/repository treat everything as "Neatoo" handwaving over the fact that it's neatoo and remotefactory (with the exception of a single page explaining the ability of remotefactory to run seperate). However, in the Neatoo repository the /docs will be Neatoo specific and in the RemoteFactory repository the /docs will be RemoteFactory specific.

## TODO 4: Package Installation Guidance

Do the setup pages (`_pages/setup/client.md` and `_pages/setup/server.md`) need updates to clarify:
- When to install `Neatoo.RemoteFactory` only
- When to install `Neatoo` (which includes RemoteFactory)
- Different setup patterns for each scenario

**Answer:**

No. For this site assume they are using Neatoo and treat Neatoo + RemoteFactory as "Neatoo". 

## TODO 5: Example Project Links

Should the example project links throughout the documentation be verified and potentially updated to align with the RemoteFactory vs Neatoo distinction?

Current examples reference:
- https://github.com/NeatooDotNet/Neatoo/tree/main/src/Examples/Person

**Answer:**

Yes
