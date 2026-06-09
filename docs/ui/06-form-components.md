# Form Components

All form inputs must support uncontrolled and controlled React states, with absolute focus-visible accessibility.

## Primitives
* **TextInput**, **PasswordInput**, **TextArea**.
* **SelectDropdown**: Supports searchable lists.
* **DatePicker**, **TimePicker**, **DateRangePicker**: ISO-8601 strict bindings.

## Domain Specific Inputs
* **RoleSelector**: Global system roles.
* **ClubAssociationSelector**: Multi-select dropdown linking an event to multiple clubs (Event ↔ Multiple Clubs).
* **ParticipantRoleSelector**: Inline dropdown modifying `event_registrations.participation_role`.
* **CapacityControl**: Numeric stepper preventing negative numbers.

## Rich Media
* **RichTextEditor**: Markdown-based editor for event descriptions.
* **FileUploadDropzone**: Integrates directly with File Storage (TBD), featuring progress bars and MIME type validation.
