**Type:** Break-Fix

# Add integration button greyed out when Fleet is off in space

**Summary:** On Elastic Cloud, the "Add Tenable Vulnerability Management" button was greyed out for a superuser. Enabling the Fleet feature for the Kibana space re-enabled it. Other integrations may behave the same way.

## Problem

In Kibana, under **Fleet > Integrations**, the integration page for Tenable Vulnerability Management (Elastic Agent, package v3.6.2) loads normally, but the **+ Add Tenable Vulnerability Management** button is greyed out and cannot be clicked.

Granting the user additional Kibana roles (for example `kibana-admin`) did not make the button available, and neither did clearing the browser cache or using a private window.

## Environment

- Product: Kibana / Fleet (integrations)
- Deployment type: Elastic Cloud Hosted
- Stack version: 8.13 (as recorded on the case)
- Integration: Tenable Vulnerability Management v3.6.2

## Cause

In this case, the Kibana space the customer was working in did not have the Fleet feature enabled. The user already had the `superuser` role (confirmed with `GET /_security/_authenticate`), so insufficient privileges did not explain the greyed-out button. Role assignment alone does not help when the feature is not available in the space.

The more common reason for a greyed-out **Add** button is a user with only `READ` access to Fleet integrations. Rule that out first (see Resolution step 1).

## Resolution

1. Confirm the user's effective privileges. Run `GET /_security/_authenticate` and check the returned roles. If the user does not have `ALL` on Fleet/Integrations (for example through `superuser`), grant it. See the related article on Fleet integration privilege levels.
2. If the user already has sufficient privileges, check whether the Fleet feature is enabled for the Kibana space you are working in, and enable it if it is not.
3. Reload Kibana and open the integration page again. The **Add** button should now be active.

In this case, enabling Fleet for the affected space activated the button, and the customer confirmed they could then complete adding the integration.

## References

- [Security privileges for the Integrations UI](https://www.elastic.co/docs/deploy-manage/users-roles/cluster-or-deployment-auth/kibana-privileges)
- [Kibana role management](https://www.elastic.co/docs/deploy-manage/users-roles/cluster-or-deployment-auth/defining-roles#roles-management-ui)
- Related KB: Can a User Use an Integration Without Having the Rights to Update or Install Them (https://support.elastic.co/knowledge/c27e90c6)

{elastic-private-context}

- Case: 02132645
- The case first treated this as a privileges problem (kibana-admin grant, screenshots, then a call). The final fix was recorded only in the closing message: "enabling Fleet for the space activated the Tenable Vulnerability Management button". The exact Kibana setting used on the call (likely Stack Management > Spaces > the space's feature visibility) was not recorded in the case. Verify before publishing and make step 2 specific.
- Related published KB c27e90c6 covers READ vs ALL privilege levels only; this article covers the space feature cause. Consider cross-linking.
- Internal: https://support.elastic.dev/knowledge/view/c27e90c6

{/elastic-private-context}
