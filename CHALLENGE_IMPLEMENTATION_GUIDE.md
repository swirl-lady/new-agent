# Auth0 AI Agents Challenge - Implementation Guide

## Quick Start

```bash
# Install dependencies
npm install

# Start Postgres database
docker compose up -d

# Run migrations (includes new tables: workspaces, audit_logs, playbooks)
npm run db:migrate

# Initialize FGA store with updated model (includes workspace support)
npm run fga:init

# Start development server
npm run dev
```

## What's New

### 1. Mission Control Dashboard (`/mission-control`)

**Purpose:** Comprehensive audit trail showcasing Auth0's identity context in every agent action.

**Features:**
- Real-time dashboard with success/failure/pending statistics
- Timeline view of all tool calls with full provenance
- Risk assessment display (low/medium/high)
- Input/output inspection for debugging
- Auth0 user context (userId, email) on every entry
- Duration metrics for performance monitoring

**Implementation:**
- Database: `audit_logs` table
- Backend: `src/lib/audit/logger.ts` - `AuditLogger` class
- Frontend: `src/app/mission-control/page.tsx`
- Actions: `src/lib/actions/audit.ts`

**Try it:**
1. Have a conversation with the agent (use Gmail, Calendar, or other tools)
2. Navigate to http://localhost:3000/mission-control
3. See every action logged with risk scores and Auth0 context

---

### 2. Risk-Adaptive Authorization

**Purpose:** Assess risk for every tool call and require step-up auth for sensitive operations.

**Features:**
- Automatic risk scoring based on:
  - Tool sensitivity (Gmail, shopping marked as high-risk)
  - Action types (send, purchase, delete, share)
  - Transaction values (purchases > $500)
  - Bulk operations (recipients > 10)
- Risk levels: low (< 40), medium (40-69), high (70+)
- Step-up authentication triggers for high-risk actions
- All assessments logged to audit trail

**Implementation:**
- Backend: `src/lib/risk/assessor.ts`
- Function: `assessRisk(toolName, args, context)`
- Returns: `{ level, score, factors, requiresStepUp }`

**Try it:**
```typescript
import { assessRisk } from '@/lib/risk/assessor';

// Low risk
const risk1 = await assessRisk('getUserInfoTool', {}, { userId: 'user123' });
// risk1.level === 'low', risk1.requiresStepUp === false

// High risk
const risk2 = await assessRisk('shopOnlineTool', { product: 'laptop', priceLimit: 1200 }, { userId: 'user123' });
// risk2.level === 'high', risk2.requiresStepUp === true
```

---

### 3. Workspaces (`/workspaces`)

**Purpose:** Multi-context personas with Auth0 FGA isolation.

**Features:**
- Users create separate contexts (Work, Personal, Family)
- Each workspace has isolated settings and future document scoping
- Auth0 FGA enforces access boundaries
- Default workspace auto-created on first use

**Implementation:**
- Database: `workspaces` table
- FGA model: `workspace` type with `owner` and `member` relations
- Backend: `src/lib/actions/workspaces.ts`
- Frontend: `src/app/workspaces/page.tsx`
- Helpers: `src/lib/workspaces/helpers.ts`

**FGA Model Update:**
```typescript
{
  type: 'workspace',
  relations: {
    can_access: {
      union: {
        child: [
          { computedUserset: { relation: 'owner' } },
          { computedUserset: { relation: 'member' } }
        ]
      }
    },
    owner: { this: {} },
    member: { this: {} }
  }
}
```

**Try it:**
1. Navigate to http://localhost:3000/workspaces
2. See existing workspaces (none by default)
3. Future: Create new workspace with name, description, color

---

### 4. Automation Playbooks

**Purpose:** Recurring agent workflows with Auth0-secured delegation.

**Status:** Database schema ready, UI in progress

**Features:**
- Define recurring automations ("Every Friday email status report")
- Specify allowed tools per playbook
- Schedule with cron expressions or manual triggers
- Inherit user's Auth0 credentials and FGA permissions
- Track execution count and last/next run times

**Implementation:**
- Database: `playbooks` table
- Backend: `src/lib/actions/playbooks.ts`
- Frontend: Coming soon

**Schema:**
```typescript
{
  id, name, description, prompt,
  schedule,        // cron expression or "manual"
  isActive,        // enable/disable
  toolsAllowed,    // array of tool names
  workspaceId,     // scope to workspace
  lastRunAt, nextRunAt, runCount,
  metadata,        // flexible JSON
  userId, userEmail, createdAt, updatedAt
}
```

**Example Playbook:**
```typescript
await createPlaybook({
  name: "Weekly Status Report",
  description: "Summarize week's activities and email manager",
  prompt: "Review this week's calendar events, Gmail threads tagged 'important', and create a status report email draft to my manager.",
  schedule: "0 16 * * 5", // Every Friday at 4pm
  toolsAllowed: ["getCalendarEventsTool", "gmailSearchTool", "gmailDraftTool"],
  workspaceId: "work-workspace-id",
  isActive: true,
  metadata: { recipients: ["manager@company.com"] }
});
```

---

## Database Schema Summary

### New Tables

```sql
-- Comprehensive audit trail
CREATE TABLE "audit_logs" (
  "id" varchar(191) PRIMARY KEY,
  "action" varchar(100) NOT NULL,
  "tool_name" varchar(100),
  "agent_role" varchar(100),
  "status" varchar(50) NOT NULL,
  "inputs" jsonb,
  "outputs" jsonb,
  "error_message" text,
  "workspace_id" varchar(191),
  "thread_id" varchar(191),
  "risk_score" varchar(50),
  "requires_approval" boolean DEFAULT false,
  "approval_status" varchar(50),
  "duration_ms" varchar(50),
  "created_at" timestamp DEFAULT now() NOT NULL,
  "user_id" varchar(191) NOT NULL,
  "user_email" varchar(191) NOT NULL
);

-- User-defined security contexts
CREATE TABLE "workspaces" (
  "id" varchar(191) PRIMARY KEY,
  "name" varchar(300) NOT NULL,
  "description" text,
  "icon" varchar(50) DEFAULT 'briefcase',
  "color" varchar(50) DEFAULT 'blue',
  "is_default" boolean DEFAULT false,
  "created_at" timestamp DEFAULT now() NOT NULL,
  "updated_at" timestamp DEFAULT now() NOT NULL,
  "user_id" varchar(191) NOT NULL,
  "user_email" varchar(191) NOT NULL
);

-- Automation definitions
CREATE TABLE "playbooks" (
  "id" varchar(191) PRIMARY KEY,
  "name" varchar(300) NOT NULL,
  "description" text,
  "prompt" text NOT NULL,
  "schedule" varchar(100),
  "is_active" boolean DEFAULT true,
  "tools_allowed" varchar(100)[],
  "workspace_id" varchar(191),
  "last_run_at" timestamp,
  "next_run_at" timestamp,
  "run_count" integer DEFAULT 0,
  "metadata" jsonb,
  "created_at" timestamp DEFAULT now() NOT NULL,
  "updated_at" timestamp DEFAULT now() NOT NULL,
  "user_id" varchar(191) NOT NULL,
  "user_email" varchar(191) NOT NULL
);
```

---

## Integration Examples

### Adding Audit Logging to a Tool

```typescript
import { AuditLogger } from '@/lib/audit/logger';
import { assessRisk } from '@/lib/risk/assessor';
import { auth0 } from '@/lib/auth0';

export const myCustomTool = tool({
  description: 'My custom tool',
  inputSchema: z.object({ query: z.string() }),
  execute: async ({ query }, context) => {
    const session = await auth0.getSession();
    const user = session?.user!;
    
    // Initialize audit logger
    const logger = new AuditLogger({
      userId: user.sub,
      userEmail: user.email!,
      workspaceId: context.workspaceId,
      threadId: context.threadId
    });
    
    // Assess risk
    const risk = await assessRisk('myCustomTool', { query }, { userId: user.sub });
    
    // Log tool start
    await logger.logToolStart('myCustomTool', { query }, 'general-agent', risk.level, risk.requiresStepUp);
    
    try {
      // Execute tool logic
      const result = await performAction(query);
      
      // Log success
      await logger.logToolSuccess({ result });
      
      return result;
    } catch (error) {
      // Log error
      await logger.logToolError((error as Error).message);
      throw error;
    }
  }
});
```

### Scoping Documents to Workspaces (Future Enhancement)

```typescript
// In documents.ts
export async function getDocumentsForWorkspace(workspaceId: string) {
  const session = await auth0.getSession();
  const user = session?.user!;
  
  // Check FGA permission
  const canAccess = await fgaClient.check({
    user: `user:${user.email}`,
    relation: 'can_access',
    object: `workspace:${workspaceId}`
  });
  
  if (!canAccess.allowed) {
    throw new Error('Access denied to workspace');
  }
  
  // Return documents scoped to workspace
  const docs = await db
    .select()
    .from(documentsTable)
    .where(eq(documentsTable.workspaceId, workspaceId));
    
  return docs;
}
```

---

## Navigation Updates

The navigation bar now includes:
- **Chat** - Main conversation interface
- **Documents** - Upload/manage/share documents
- **Mission Control** - Audit dashboard (NEW)
- **Workspaces** - Context management (NEW)

---

## Next Steps for Full Challenge Submission

### Immediate (Week 1)
- [x] Database schemas for workspaces, audit_logs, playbooks
- [x] Audit logging infrastructure
- [x] Risk assessment system
- [x] Mission Control dashboard
- [x] Workspaces page skeleton
- [ ] **Connect audit logging to existing tools**
- [ ] **Workspace creation UI**
- [ ] **Risk-based step-up auth integration**

### Near-term (Week 2)
- [ ] Enhanced integrations (Slack, Notion, GitHub)
- [ ] RAG filtering by workspace
- [ ] Playbook execution engine
- [ ] Team collaboration features
- [ ] Data lifecycle management UI

### Polish (Week 3)
- [ ] Video demo recording
- [ ] Architecture diagram
- [ ] Security white paper
- [ ] Performance benchmarks
- [ ] Blog post draft

---

## Testing Checklist

### Audit Trail
- [ ] Have conversation using multiple tools
- [ ] Navigate to Mission Control
- [ ] Verify all actions logged with timestamps
- [ ] Check risk scores displayed correctly
- [ ] Expand inputs/outputs to view details
- [ ] Confirm Auth0 context (userId, email) present

### Risk Assessment
- [ ] Test low-risk tool (getUserInfo)
- [ ] Test medium-risk tool (Gmail search)
- [ ] Test high-risk tool (shopping with high price)
- [ ] Verify risk scores calculated correctly
- [ ] Check `requiresStepUp` flag for high-risk

### Workspaces
- [ ] Navigate to /workspaces
- [ ] See empty state if no workspaces
- [ ] Future: Create new workspace
- [ ] Future: Set default workspace
- [ ] Future: Filter documents by workspace

### FGA Integration
- [ ] Run `npm run fga:init` after migration
- [ ] Verify workspace type in FGA model
- [ ] Test workspace ownership relations
- [ ] Confirm document access still works

---

## Architecture Highlights

```
User Request
    ↓
Auth0 Middleware (authentication)
    ↓
API Route /api/chat
    ↓
AI Agent (Vercel AI SDK)
    ↓
Tool Call
    ↓
Risk Assessor → Calculate risk score
    ↓
Audit Logger → Log tool_start with risk
    ↓
Auth0 FGA Check → Verify permissions
    ↓
Token Vault → Get delegated credentials
    ↓
Execute Tool
    ↓
Audit Logger → Log tool_success/error
    ↓
Stream Response to Client
    ↓
Mission Control → Display in audit timeline
```

---

## Key Files Reference

### New Files
- `src/lib/audit/logger.ts` - Audit logging class
- `src/lib/risk/assessor.ts` - Risk assessment logic
- `src/lib/actions/audit.ts` - Audit query actions
- `src/lib/actions/workspaces.ts` - Workspace CRUD
- `src/lib/actions/playbooks.ts` - Playbook management
- `src/lib/workspaces/helpers.ts` - Workspace utilities
- `src/lib/db/schema/audit-logs.ts` - Audit schema
- `src/lib/db/schema/workspaces.ts` - Workspace schema
- `src/lib/db/schema/playbooks.ts` - Playbook schema
- `src/app/mission-control/page.tsx` - Audit dashboard
- `src/app/workspaces/page.tsx` - Workspace management
- `AUTH0_CHALLENGE_FEATURES.md` - Detailed feature docs

### Modified Files
- `src/lib/fga/fga-init.ts` - Added workspace type to FGA model
- `src/app/layout.tsx` - Added Mission Control & Workspaces to nav

---

## Support & Resources

- **Challenge Page:** https://dev.to/challenges/auth0-2025-10-08
- **Auth0 AI Docs:** https://auth0.com/ai/docs
- **FGA Docs:** https://auth0.com/fine-grained-authorization
- **Vercel AI SDK:** https://sdk.vercel.ai/docs

---

## License

MIT License - See LICENSE file
