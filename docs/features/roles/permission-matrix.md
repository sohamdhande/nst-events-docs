# Global Permission Matrix

This matrix is the foundation for Row Level Security (RLS), dashboard permissions, and backend authorization.

| Action | Student | Member | Core Member | Club Admin | Faculty Mentor | Faculty Admin | Platform Admin |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| **Authentication & Profile** |
| Login via OAuth | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Update own profile | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| View public profiles | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Events (Public & Reading)** |
| View public events | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| View private club events | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Register for event | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ |
| Form/Join a team | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ |
| **Event Management** |
| Create Draft Event | ❌ | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Edit Draft Event | ❌ | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Submit for Approval | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ | ✅ |
| Approve Event | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ |
| Publish Event | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ |
| Archive Event | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ | ✅ |
| Cancel Event | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ | ✅ |
| **Attendance** |
| Scan QR / Mark Attendance| ✅ | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ |
| Generate Event QR | ❌ | ❌ | ✅ | ✅ | ❌ | ❌ | ✅ |
| View Event Attendance | ❌ | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Modify/Override Attendance| ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |
| Export Attendance CSV | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ | ✅ |
| **Club Management** |
| View Club Roster | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Add Member to Club | ❌ | ❌ | ✅ | ✅ | ❌ | ❌ | ✅ |
| Remove Member | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ | ✅ |
| Promote to Core Member | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ | ✅ |
| Edit Club Profile | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ | ✅ |
| Delete Club | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |
| **Analytics & Reporting** |
| View Event Analytics | ❌ | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ |
| View Club Analytics | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ | ✅ |
| View Campus Analytics | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ |
| **Leaderboard** |
| View Individual Leaderboard| ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| View Club Leaderboard | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Manually Award Points | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |
| **System Settings** |
| Manage Platform Settings | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |
| View Audit Logs | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |
| Manage RBAC Roles | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |
