# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is an AI Agent development workflow repository that uses a structured, phase-based approach to generate multi-agent systems. The workflow progresses through 6 distinct phases from business requirements to code generation, producing detailed design documents and executable Python code for AI agent applications.

### Key Components

- **Workflow Engine**: `.claude/rules/WORKFLOW.md` and `artifact-catalog.yaml` define the artifact generation pipeline
- **Phase-based Skills**: `.claude/skills/phase-0X-*/` contain prompts, templates, and references for each development phase
- **Artifact Outputs**: `artifacts/0X_*/outputs/` store generated design documents and code
- **Progress Tracking**: `STATE.md` maintains current phase status and completion state
- **Generated Code**: `artifacts/06_code-generation/src/` contains the final multi-agent Python application

### Architecture

The system implements a **multi-agent orchestration pattern** using Amazon Bedrock and the Strands Agents framework:

1. **Orchestrator Agent** (申請受付窓口エージェント) - Routes user requests to specialist agents
2. **Transport Application Agent** (交通費精算申請エージェント) - Handles transportation expense claims
3. **Expense Application Agent** (経費精算申請エージェント) - Handles general expense claims

The agents use:
- **Tools**: Transport cost calculation, Excel application generation
- **Session Management**: File-based session storage with invocation state
- **Hooks**: Loop control (prevents infinite loops), human approval (HITL), error handling
- **Knowledge**: Business rules and policies loaded dynamically into system prompts
- **Guardrails**: Content filtering using Amazon Bedrock Guardrails (PII, harmful content)

## Working with the Workflow

### Critical Rules - READ BEFORE ANY WORK

1. **NEVER modify `.claude/rules/` files** unless explicitly instructed by the user. These are steering files managed by the user.

2. **ALWAYS check STATE.md first** at session start:
   - Read `STATE.md` in the project root
   - Check "次アクション待ち状態" (Next Action Status)
   - If status is `⏸️ ユーザー指示待ち` → **STOP** and ask user for instructions
   - If status is `▶️ 作業中` → Proceed with the next incomplete artifact

3. **NEVER auto-advance between phases** - When a phase completes, display:
   ```
   ✅ [Phase Name] が完了しました。
   次フェーズ（[Next Phase Name]）を開始する場合は、その旨を指示してください。
   ```
   Wait for explicit user instruction before starting the next phase.

4. **Phase Multi-Round Workflow**:
   - Round 1: Create all artifacts in sequence per `artifact-catalog.yaml` `default_sequence`
   - If any artifact has `depends_on` pointing to same-phase artifacts not yet created, write `要件上未定義` for those sections
   - After Round 1 completes, check if Round 2 is needed (see WORKFLOW.md 依存ファイルが存在しない場合の例外ルール)
   - Round 2+: Fill in `要件上未定義` sections using now-available dependency artifacts

### Artifact Generation Workflow

For each artifact, follow this sequence:

1. **Load References**: Read from `artifact-catalog.yaml`:
   - The artifact's `prompt` file (generation instructions)
   - The artifact's `template` file (output structure)
   - All `depends_on` artifacts (input data)

2. **Dependency Check**: Before starting work:
   - Search all dependency artifacts for `要件上未定義`, `〇〇フェーズで定義`, `TBD`
   - If found → **STOP** and report to user (do not proceed even if "minor")
   - Exception: Round 1 creation when dependency files don't exist yet

3. **Generate**: 
   - Follow the prompt instructions exactly
   - Use the template structure completely (no sections omitted)
   - Output in Japanese (except code identifiers which use English)
   - Code comments must be in Japanese

4. **Update STATE.md**:
   - Before starting: Change artifact status to `🔄 作業中`
   - After completion: Change to `✅ 完了`
   - After all artifacts in phase: Update phase completion status

5. **Phase Quality Check**: After ALL artifacts in a phase are complete, run the phase quality check defined in WORKFLOW.md (do not run quality checks during individual artifact creation).

### Development Phases

The workflow consists of 6 sequential phases:

1. **01_business-requirements**: Business requirements, processes, use cases (BR-01 ~ BR-07)
2. **02_system-requirements**: Functional requirements, agents, tools, data models (SR-01 ~ SR-19)
3. **03_system-design**: System design, session management, error handling (SD-01 ~ SD-10)
4. **04_basic-design**: Tool, data model, agent, handler basic design (BD-01 ~ BD-05)
5. **05_detailed-design**: Detailed design for all components (DD-01 ~ DD-09)
6. **06_code-generation**: Implementation tasks, code generation, testing (IG-01 ~ IG-04)

### Using Phase Skills

To execute a complete phase, use the corresponding skill:

```
/phase-01-business-requirements   # Start Phase 1
/phase-02-system-requirements     # Start Phase 2
/phase-03-system-design           # Start Phase 3
/phase-04-basic-design            # Start Phase 4
/phase-05-detailed-design         # Start Phase 5
/phase-06-code-generation         # Start Phase 6
```

Each skill will automatically:
- Load artifact definitions from `artifact-catalog.yaml`
- Follow the `default_sequence` order
- Apply prompts and templates for each artifact
- Update `STATE.md` with progress
- Run phase quality checks after all artifacts complete

## Working with Generated Code

### Directory Structure (Rule R1)

The generated code follows a strict directory structure in `artifacts/06_code-generation/src/`:

```
src/
├── agents/          # *_agent.py - Agent implementations
├── config/          # *_config.py, settings.py - Configuration
├── data/            # *.json - Master data files
├── evals/           # eval_*.py - Evaluation tests
├── guardrails/      # *.yaml - Guardrail definitions
├── handlers/        # *_handler.py, *_hook.py - Handlers and hooks
├── knowledge/       # *_policies.py - Business rules
├── logs/            # Log files (runtime)
├── models/          # data_models.py - Pydantic models
├── output/          # Generated Excel files (runtime)
├── prompt/          # prompt_*.py - System prompts
├── session/         # session_manager.py - Session management
├── storage/         # Session storage (runtime)
├── template/        # *.xlsx - Excel templates
├── tests/           # Unit and integration tests
│   ├── unit/
│   └── integration/
└── tools/           # *_tools.py, *_generator.py - Agent tools
```

**NEVER create files outside these directories or add new top-level directories without explicit user approval.**

### Running Tests

```bash
# Run all tests with coverage
cd artifacts/06_code-generation/src
pytest --cov=. --cov-report=term-missing

# Run specific test categories
pytest tests/unit/                    # Unit tests only
pytest tests/integration/             # Integration tests only
pytest tests/unit/test_transport_tools.py  # Specific test file

# Run with verbose output
pytest -v
```

### Running the Application

```bash
# Set up environment
cd artifacts/06_code-generation/src
cp .env.template .env
# Edit .env with your AWS credentials and Bedrock configuration

# Install dependencies
pip install -r ../../../requirements.txt

# Run the application
python main.py
```

The application will:
1. Prompt for applicant name (申請者名)
2. Start interactive dialogue loop
3. Route requests to appropriate specialist agents
4. Generate Excel application forms in `output/` directory

Type `終了`, `exit`, or `quit` to terminate.

### Key Configuration Files

- **`.env`**: AWS credentials, Bedrock model ID, Guardrail ID
- **`src/config/settings.py`**: Agent behavior settings (max iterations, window size, deadlines)
- **`src/config/model_config.py`**: Bedrock model configuration
- **Environment variable prefixes**:
  - `ECAAS_RECEPTION_*` - Orchestrator agent settings
  - `ECAAS_TRANSPORT_*` - Transport agent settings
  - `ECAAS_EXPENSE_*` - Expense agent settings

### Modifying Business Rules

Business rules are defined in:
- `src/knowledge/transport_policies.py` - Transportation expense rules
- `src/knowledge/expense_policies.py` - General expense rules

These are loaded into agent system prompts dynamically. After modification, restart the application.

### Adding New Tools

1. Define tool in `src/tools/`:
   - Use `@tool` decorator from `strands_agents`
   - For tools needing session context, use `@tool(context=True)` and accept `ToolContext`
   - Return `dict` with `success`, `message`, and result data
   - Handle `ValidationError` and `Exception`

2. Update agent configuration to include new tool in its tools list

3. Update system prompt to instruct agent when to use the tool

4. Write unit tests in `tests/unit/test_<tool_name>.py`

## Common Development Tasks

### Add a New Artifact Type

1. **User must update** `artifact-catalog.yaml` with new artifact definition (id, name, phase, order, depends_on, prompt, template, output)
2. Create prompt file in `.claude/skills/phase-XX-*/references/`
3. Create template file in `.claude/skills/phase-XX-*/assets/`
4. Update phase's `default_sequence` in `artifact-catalog.yaml`
5. Run phase skill to generate

### Debug Agent Loops

The system has loop control to prevent infinite loops:

- Check `logs/app.log` for loop count messages
- Default max iterations: 10 (configurable in `settings.py`)
- `LoopLimitError` is raised when exceeded
- Adjust `max_iterations` in agent settings if legitimate use case needs more loops

### Debug Human Approval Flow

- Approval is required for: `generate_transport_application`, `generate_expense_application`
- Console will display application summary and prompt: `承認しますか？ (OK/修正を要求/キャンセル)`
- Input `OK` to approve, `キャンセル` to cancel, or modification request text
- Check `tests/integration/test_multi_agent_flow.py` for approval flow examples

## Language Conventions

Design documents are 100% Japanese. In code, identifiers (variables, functions, classes) use English; everything else (comments, docstrings, log messages, error messages, user-facing output) uses Japanese.

## Dependencies

Core dependencies (see `requirements.txt`):
- `strands-agents>=1.0.0` - Multi-agent framework
- `strands-agents-tools>=0.4.0` - Agent tool utilities  
- `boto3>=1.34.0` - AWS SDK (for Bedrock)
- `pydantic>=2.4.0` - Data validation
- `pytest>=8.0.0` - Testing framework
- `openpyxl>=3.1.0` - Excel file generation

