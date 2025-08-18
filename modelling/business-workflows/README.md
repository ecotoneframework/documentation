# Workflows

### 🔄 **Use [Handler Chaining](connecting-handlers-with-channels.md) when:**
- You have simple, linear workflows
- Steps are tightly coupled to specific workflows
- You don't need dynamic workflow construction

### 📊 **Use [Sagas](sagas.md) when:**
- You need to **remember state** between steps
- Workflows span **long periods** (hours, days, weeks)
- You need **compensation logic** for failures
- Workflows involve **human interaction** or external approvals

### ✅ **Use Orchestrator when:**
- You have **predefined workflows** with clear steps
- Steps need to be **reusable** across different workflows
- You need **dynamic workflow construction**
- Workflow logic is **separate from step implementation**