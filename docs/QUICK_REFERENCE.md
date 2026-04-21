# Quick Reference: Scripts and Multi-Asset Workstreams

## 🎯 What Was Added

This PR introduces:
1. **Scripts folder structure** (common + skill-specific)
2. **Complete example workstream** (data-analytics) with all asset types
3. **Comprehensive documentation** on how to structure and develop multi-asset workstreams

---

## 📚 Documentation

### Main Guides

| Guide | Purpose | Link |
|---|---|---|
| **Scripts Guide** | Scripts folder structure, types, and best practices | [docs/scripts-guide.md](docs/scripts-guide.md) |
| **Multi-Asset Workstreams** | How to build complete workstreams with multiple assets | [docs/multi-asset-workstreams.md](docs/multi-asset-workstreams.md) |
| **Data Analytics Example** | Reference implementation of multi-asset workstream | [workstreams/data-analytics/](workstreams/data-analytics/) |

### Key Documents

- **Workflow Documentation**: [complete-data-analysis-workflow.md](workstreams/data-analytics/workflows/complete-data-analysis-workflow.md)
- **Workstream README**: [data-analytics/README.md](workstreams/data-analytics/README.md)

---

## 🏗️ Scripts Folder Structure

### Two Types of Scripts

**Common Scripts** (`shared/scripts/`)
- Used by multiple workstreams
- Foundational capabilities (e.g., data cleaning, validation)
- Example: `data-cleaner.md`

**Skill-Specific Scripts** (`workstreams/<name>/scripts/`)
- Tailored to specific workstream needs
- Implements workstream-specific logic
- Example: `csv-to-json-transformer.md`

### Decision Tree

```
Is the script useful to multiple workstreams?
├─ YES → Place in shared/scripts/
└─ NO → Place in workstreams/<name>/scripts/
```

---

## 🎨 Data Analytics Workstream (Example)

### Assets Included

| Type | Asset | Purpose |
|---|---|---|
| Script (common) | `data-cleaner.md` | Clean and validate data |
| Script (skill-specific) | `csv-to-json-transformer.md` | Convert CSV to JSON |
| Prompt | `statistical-analysis-prompt.md` | Generate statistics |
| Skill | `sales-trend-analyzer.md` | Analyze trends & forecast |
| Agent | `data-insights-agent.md` | Orchestrate complete pipeline |
| Workflow | `complete-data-analysis-workflow.md` | Document end-to-end process |

### Pipeline Flow

```
Raw CSV file
    ↓
csv-to-json-transformer (transform)
    ↓
data-cleaner (clean & validate)
    ↓
statistical-analysis-prompt (analyze)
    ↓
sales-trend-analyzer (trends & forecast)
    ↓
data-insights-agent (automate all)
```

### Usage Modes

**Manual**:
```bash
python workstreams/data-analytics/scripts/csv_to_json.py --input data.csv --output /tmp/data.json
python shared/scripts/data_cleaner.py --input /tmp/data.json --output /tmp/clean.json
# Then use prompts/skills manually
```

**Fully Automated**:
```
@workspace You are a data insights agent.
Data source: data/sales.csv
Analysis type: sales-trends
Context: Q1 e-commerce
```

---

## 🚀 Quick Start

### Explore the Example

1. **Read the overview**: `workstreams/data-analytics/README.md`
2. **Study the complete workflow**: `workstreams/data-analytics/workflows/complete-data-analysis-workflow.md`
3. **Review individual assets**:
   - Scripts: Check both common and skill-specific scripts
   - Prompt: See how statistical analysis works
   - Skill: Understand focused capability (sales trends)
   - Agent: See autonomous orchestration

### Build Your Own Workstream

Follow the guide in `docs/multi-asset-workstreams.md`:

1. **Plan**: Define problem and end-to-end process
2. **Build bottom-up**:
   - Scripts first (data prep)
   - Prompts second (LLM tasks)
   - Skills third (specialized capabilities)
   - Agents fourth (orchestration)
   - Workflows last (documentation)
3. **Document**: Integration points and dependencies
4. **Test**: Manual, semi-automated, and fully automated paths
5. **Register**: Add to `workstreams/registry.json`

---

## 📝 Key Concepts

### Asset Composition

**Scripts** → Prepare data for LLM consumption  
**Prompts** → Provide templates for specific tasks  
**Skills** → Focused capabilities built on prompts  
**Agents** → Orchestrate multiple tools/skills  
**Workflows** → Document complete processes

### Progressive Enhancement

- **Level 1**: Manual (scripts only)
- **Level 2**: Semi-automated (scripts + manual prompts)
- **Level 3**: Fully automated (agent orchestration)

### Reusability

- Common utilities in `shared/scripts/`
- Workstream-specific in `workstreams/<name>/scripts/`
- Clear dependency documentation

---

## 🔍 Where to Find Things

### Scripts
- Common: `shared/scripts/`
- Skill-specific: `workstreams/<name>/scripts/`

### Documentation
- Scripts guide: `docs/scripts-guide.md`
- Multi-asset workstreams: `docs/multi-asset-workstreams.md`

### Example Workstream
- All assets: `workstreams/data-analytics/`
- Complete workflow: `workstreams/data-analytics/workflows/complete-data-analysis-workflow.md`

### Templates
- Script template: `templates/script-template.md`
- All templates: `templates/`

---

## ✅ Best Practices

### Scripts
- ✅ Support both CLI flags and environment variables
- ✅ Document security considerations
- ✅ Include complete working implementation
- ✅ Provide realistic examples
- ✅ Make idempotent (safe to re-run)

### Multi-Asset Workstreams
- ✅ Build bottom-up (scripts → prompts → skills → agents)
- ✅ Provide multiple execution paths (manual, semi-auto, auto)
- ✅ Document integration points clearly
- ✅ Test each asset independently
- ✅ Create comprehensive workflow documentation

### Documentation
- ✅ Follow template structure
- ✅ Include examples with real data
- ✅ Document dependencies explicitly
- ✅ Show pipeline position
- ✅ Add security notes

---

## 📊 Statistics

### New Assets Created
- 6 complete assets (2 scripts, 1 prompt, 1 skill, 1 agent, 1 workflow)
- 2 comprehensive guides (15,000+ words)
- 1 new workstream registered

### Documentation Coverage
- Complete implementation code for all scripts
- Multiple usage examples per asset
- End-to-end workflow documentation
- Security considerations for all assets

---

## 🎓 Learning Path

### For Beginners
1. Read: `workstreams/data-analytics/README.md`
2. Explore: Individual assets in order (scripts → prompts → skills)
3. Try: Run manual pipeline with sample data

### For Intermediate Users
1. Study: `workstreams/data-analytics/workflows/complete-data-analysis-workflow.md`
2. Understand: Asset composition patterns
3. Build: Create your own skill using existing scripts

### For Advanced Users
1. Review: `docs/multi-asset-workstreams.md`
2. Design: Plan a new multi-asset workstream
3. Implement: Build complete pipeline with agent orchestration

---

## 💡 Tips

- Start with scripts — they're the foundation
- Use the data-analytics workstream as a template
- Document as you build, not after
- Test each component independently
- Provide both manual and automated paths
- Keep assets focused and composable

---

## 🆘 Common Questions

**Q: When should I create a script vs a prompt?**  
A: Scripts for data transformation/processing; prompts for LLM instructions.

**Q: Where should I put a script that's used by one workstream?**  
A: In `workstreams/<name>/scripts/` (skill-specific).

**Q: Where should I put a script that's used by multiple workstreams?**  
A: In `shared/scripts/` (common utility).

**Q: Do I need to create all asset types for every workstream?**  
A: No — create only what's needed for your use case.

**Q: How do I know if my workstream is complex enough for an agent?**  
A: If you have 3+ steps with decision points, consider an agent.

---

## 📞 Next Steps

1. ✅ Review this PR and provide feedback
2. ✅ Explore the data-analytics workstream
3. ✅ Read the comprehensive guides
4. ✅ Build your own multi-asset workstream using these patterns

Happy building! 🚀
