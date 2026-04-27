# MathQuiz AI - Cursor Rules

## Identity
- Project: `MathQuiz AI`
- Product: Vietnamese math exam generator and online quiz system
- Main app directory: `app/`
- Source of truth: `plan/init.md`, `plan/update1.md`

## Tech Stack
- Next.js 16 App Router
- TypeScript
- React 19
- Tailwind CSS v4
- KaTeX for math rendering
- Zod for schemas and validation
- `ai` and `@ai-sdk/google` for AI integration

## Repo Structure
- Routes and app shell: `app/src/app`
- UI components: `app/src/components`
- Business logic and helpers: `app/src/lib`
- Do not invent a different architecture unless explicitly requested

## Communication
- Always reply in Vietnamese unless the user asks otherwise
- Keep common technical terms in English
- Be concise, practical, and tied to the actual codebase
- Inspect the repo before asking clarifying questions

## Core Working Rules
- Follow the current codebase, not generic assumptions
- Keep MVP behavior stable unless the task explicitly changes scope
- Be honest about partial implementations
- Do not describe mock features as completed production features
- Reuse existing utilities, schemas, and patterns before adding new ones

## Product Reality
- AI generation is not fully production-ready if code still uses mock data
- OCR is not fully implemented if flow still depends on mocked parsing
- Weakness analysis may be heuristic and client-side unless server or AI logic is added
- `app/README.md` is boilerplate and not the product spec

## Architecture Rules
- Put UI rendering in components
- Put domain logic, transformations, grading, persistence helpers, and AI helpers in `lib`
- Validate API input and output with Zod where appropriate
- Keep components focused and avoid mixing UI with heavy business logic
- Preserve quiz flow: config -> generate -> quiz -> result -> weakness / OCR review

## UI Rules
- Preserve the existing visual language unless redesign is requested
- Keep desktop and mobile behavior working
- Keep KaTeX output readable
- Do not break keyboard shortcuts, timer flow, or answer selection UX

## Anti-Cheat Rules
- Preserve question and option randomization
- Preserve timer and submit behavior
- Preserve tab-switch tracking and warnings unless explicitly changing that behavior
- Preserve quiz session persistence unless replacing it with a better verified solution

## Code Style
- Variables and functions: `camelCase`
- Components and types: `PascalCase`
- Constants: `SCREAMING_SNAKE_CASE`
- Follow existing file naming patterns in this repo
- Prefer simple and readable code over clever abstractions
- Add comments only when needed for non-obvious logic
- Avoid new dependencies unless clearly justified

## Workflow
- Read relevant files first
- For non-trivial tasks, form a short plan before editing
- Make the smallest correct change that fits the current architecture
- After changes, run the smallest useful verification available
- If tests do not exist, give manual verification steps

## Validation Priorities
- Config form validation
- `/api/generate`
- Quiz timer and submit flow
- Grading correctness
- Weakness analysis behavior
- OCR upload and review flow

## Git Safety
- Never auto-push
- Never commit unless explicitly asked
- Do not rewrite history unless explicitly asked
- Avoid destructive commands
- Respect unrelated user changes in the workspace

## Next.js Warning
- This project uses a newer Next.js version
- Verify framework-specific behavior from the local repo and current docs before changing APIs or conventions
