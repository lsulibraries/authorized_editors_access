# Authorized Editors Access

This custom Drupal 11 module grants per-node **edit** (update) access to users listed in the `field_authorized_editors` user reference field on a node.

It is designed for cases where content follows normal editorial workflows, but selected users need update access to specific nodes without giving them broad content-type permissions.

## What it does

The module implements Drupal’s node access hook so that when Drupal checks whether a user can **update** a node, it also looks at `field_authorized_editors`.

- If the current user’s account is referenced in `field_authorized_editors`, the module **allows** the update operation for that node.  
- For all other operations (`view`, `delete`, etc.), the module stays **neutral**, leaving access to core and any other access modules.

This approach does **not** write to the `node_access` table and does **not** alter view access for anonymous or other visitors. Public viewing continues to be controlled by normal role permissions such as **View published content** and any existing view‑level access solutions.

## File structure

Place the module in the custom modules directory using this structure:

```text
web/
├── modules/
│   └── custom/
│       └── authorized_editors_access/
│           ├── authorized_editors_access.info.yml
│           └── authorized_editors_access.module
```

- `authorized_editors_access.info.yml` tells Drupal about the module and its dependencies.  
- `authorized_editors_access.module` contains the node access hook implementation.

No `.install` file is required for this behavior.

## Example files

### `authorized_editors_access.info.yml`

```yaml
name: Authorized Editors Access
type: module
description: Grants per-node edit access to users listed in field_authorized_editors.
core_version_requirement: ^11
package: Custom
dependencies:
  - drupal:node
  - drupal:user
```

### `authorized_editors_access.module`

```php
<?php

declare(strict_types=1);

use Drupal\Core\Access\AccessResult;
use Drupal\Core\Session\AccountInterface;
use Drupal\node\NodeInterface;

/**
 * Implements hook_node_access().
 *
 * Grants per-node update access to users referenced in field_authorized_editors.
 * View and delete access are left to core and other modules.
 *
 * @param \Drupal\node\NodeInterface $node
 *   The node being accessed.
 * @param string $op
 *   The operation: 'view', 'update', 'delete', etc.
 * @param \Drupal\Core\Session\AccountInterface $account
 *   The user account.
 *
 * @return \Drupal\Core\Access\AccessResultInterface
 *   The access result.
 */
function authorized_editors_access_node_access(NodeInterface $node, $op, AccountInterface $account) {
  // Only alter UPDATE access. Leave view/delete to core and other modules.
  if ($op !== 'update') {
    return AccessResult::neutral();
  }

  // Admins or roles with global edit permissions should be handled elsewhere.
  // Stay neutral so we don't override broader permissions.
  if ($account->hasPermission('administer nodes') || $account->hasPermission('bypass node access')) {
    return AccessResult::neutral();
  }

  // If the node does not have the field, or it is empty, do nothing.
  if (!$node->hasField('field_authorized_editors') || $node->get('field_authorized_editors')->isEmpty()) {
    return AccessResult::neutral();
  }

  $target_ids = $node->get('field_authorized_editors')->getValue();
  $authorized_uids = array_map(static function (array $item): int {
    return (int) $item['target_id'];
  }, $target_ids);

  // If current user is listed as an authorized editor, allow update.
  if (in_array((int) $account->id(), $authorized_uids, true)) {
    return AccessResult::allowed()
      ->cachePerUser()
      ->addCacheableDependency($node);
  }

  // Otherwise, remain neutral so other access logic can decide.
  return AccessResult::neutral();
}
```

This implementation does not register any node access realms or write node access records, so it cannot accidentally block anonymous viewing of public nodes.

## Installation

1. Create the folder:  

   `web/modules/custom/authorized_editors_access`

2. Add the `.info.yml` and `.module` files shown above into that folder.

3. Enable the module in the admin UI:  
   - Go to **Extend** (`/admin/modules`).  
   - Find **Authorized Editors Access**.  
   - Check it and click **Install**.

   Or enable with Drush:

   ```bash
   drush pm:enable authorized_editors_access
   ```

Because this version does not use the node grants / `node_access` table, a `node_access_rebuild()` is **not** required just for enabling or disabling the module.

## Behavior and scope

- Grants **update** access per node to users listed in `field_authorized_editors`.  
- Does **not** change view or delete access. Those are still determined by:  
  - core permissions (e.g., **View published content**), and  
  - any other node access or view‑level modules installed.

Revision access is controlled separately by Drupal’s revision permissions (such as **View revisions** per bundle or **View all revisions**) and is not automatically granted by this module.

## Important notes

- This module only ever **adds** an extra way to allow updates. It does **not** add any explicit denies.
- If you previously used a node‑grants‑based version that wrote to `node_access` and saw “Access denied” for anonymous users, make sure that version is removed/uninstalled and a node access rebuild has been run before using this simplified version.
- Make sure `field_authorized_editors` is a user reference field on the content type(s) where you want this behavior. If the field is missing or empty, the module has no effect.

## Suggested testing

After enabling the module, test at least these scenarios:

- A user listed in `field_authorized_editors` can edit the node, even if they do not have “edit any” permissions for that content type.  
- A user not listed in the field cannot edit the node, unless they already have global edit permissions via roles.  
- Anonymous users and normal visitors can still view published pages as before (assuming **View published content** is granted to their role).  
- Revision access behaves according to your revision permissions, unchanged by this module.

## Next improvements

Potential enhancements:

- Restrict the behavior to specific content types by checking `$node->bundle()` inside the access hook.
- Add automated tests to confirm update access is granted/denied correctly for different roles and field values.
- Introduce a configuration form if you want to make the field name or affected bundles configurable instead of hard-coded.