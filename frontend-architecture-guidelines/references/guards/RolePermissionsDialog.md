# RolePermissionsDialog (`RolePermissionsDialog.tsx`)

Located at: `src/components/guards/RolePermissionsDialog.tsx` or `src/routes/settings/access/-components/RolePermissionsDialog.tsx`

## Purpose

A modal dialog that manages permission selection state for a specific role and executes the API assignment mutation.

---

## Implementation

```tsx
import { useState, useEffect } from "react";
import { Dialog, DialogContent, DialogTitle } from "@/components/ui/dialog";
import { Button } from "@/components/ui/button";
import { toast } from "sonner";
import { useAssignPermissions } from "../-queries";
import { PaginatedPermissionTable } from "./PaginatedPermissionTable";
import type { Role } from "../-types";

interface RolePermissionsDialogProps {
  role: Role;
  isOpen: boolean;
  onClose: () => void;
}

export function RolePermissionsDialog({
  role,
  isOpen,
  onClose,
}: Readonly<RolePermissionsDialogProps>) {
  const assignPermissionsMutation = useAssignPermissions(role.id);

  const [selectedPermissions, setSelectedPermissions] = useState<Set<number>>(
    new Set(),
  );

  useEffect(() => {
    if (isOpen) {
      setSelectedPermissions(new Set(role.permissions || []));
    }
  }, [isOpen, role]);

  const handlePermissionToggle = (id: number, checked: boolean) => {
    const next = new Set(selectedPermissions);
    if (checked) next.add(id);
    else next.delete(id);
    setSelectedPermissions(next);
  };

  const handleSave = async () => {
    try {
      await assignPermissionsMutation.mutateAsync(
        Array.from(selectedPermissions),
      );
      toast.success("Permissions updated successfully");
      onClose();
    } catch {
      toast.error("Failed to update permissions");
    }
  };

  return (
    <Dialog open={isOpen} onOpenChange={onClose}>
      <DialogContent className="max-w-4xl max-h-[90vh] overflow-y-auto">
        <DialogTitle>Assign Permissions to {role.name}</DialogTitle>

        <div className="mt-6">
          <PaginatedPermissionTable
            selectedIds={Array.from(selectedPermissions)}
            onToggle={handlePermissionToggle}
          />
        </div>

        <div className="mt-8 flex justify-end gap-3">
          <Button
            variant="outline"
            onClick={onClose}
            disabled={assignPermissionsMutation.isPending}
          >
            Cancel
          </Button>
          <Button
            onClick={handleSave}
            disabled={assignPermissionsMutation.isPending}
          >
            {assignPermissionsMutation.isPending ? "Saving..." : "Save Changes"}
          </Button>
        </div>
      </DialogContent>
    </Dialog>
  );
}
```
