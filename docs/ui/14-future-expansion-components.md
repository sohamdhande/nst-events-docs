# Future Expansion Components

The component architecture is designed to support future modules seamlessly.

## Certificates
* `EmptyState` and `DataTable` are ready to support a `/certificates` route.
* `DynamicQRCodeRenderer` can be adapted to render on-chain verification codes.

## Recruitment
* `RoleAssignmentPanel` forms the basis for Club Intake panels.
* `ApprovalQueueItem` can be adapted for Interview Candidate approvals.

## Media Galleries
* `Carousel` and `FileUploadDropzone` primitives exist to support event photo galleries.

## Multi-Campus
* ContextSwitchers are already built to support `active_campus_id`.

## Guest Event Passes
* **Status**: Deferred (Out of Scope for V1)
* **Architecture Impact**: Will require expanding auth beyond the institutional domain lock and creating a lightweight `Guest` role.
