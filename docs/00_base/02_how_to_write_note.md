# Dev Notes Writer Prompt

## Purpose and Background

This prompt is designed to help AI assistants create comprehensive development notes that can be used to onboard the next AI assistant seamlessly. When working on a project over multiple sessions, context is often lost between conversations. These development notes serve as a persistent memory that allows any AI to pick up exactly where the previous one left off, understanding not just what was built but why certain decisions were made.

The target audience for these notes is explicitly another AI assistant, not a human developer. This means the notes should be optimized for machine comprehension: detailed, explicit, and structured in a way that provides maximum context. Human developers may also find these notes useful, but the primary goal is AI-to-AI knowledge transfer.

## Communication Language

When communicating with the user, always use Traditional Chinese (繁體中文). However, the actual development notes document should be written in English. This is because AI models generally process English more efficiently, and the notes are intended for AI consumption. The user has explicitly requested this bilingual approach.

## File Naming Convention

Development notes should be saved in the docs directory of the relevant project. The default naming format is `yyyymmdd_xxxx.md` where yyyy is the four-digit year, mm is the two-digit month, dd is the two-digit day, and xxxx is a short descriptive identifier in lowercase English letters with underscores if needed.

For example, if today is January 16, 2026 and the work involved initializing a project, the filename would be `20260116_init_project.md`. If the work involved fixing authentication bugs, it might be `20260116_auth_bugfix.md`. The AI should analyze the conversation content and determine an appropriate descriptive identifier that captures the essence of what was accomplished.

## Execution Flow

When the user asks you to write development notes, follow this process.

First, review the entire conversation to understand what was discussed, what decisions were made, what was implemented, and what the current state of the project is. Pay special attention to any explicit user preferences or requirements that were stated, as these represent important context for future AI assistants.

Second, ask the user where they want the notes to be saved. Provide a suggested path based on the project being worked on, typically in a docs subdirectory of the project root. Also suggest a filename following the yyyymmdd_xxxx.md convention, where xxxx is your best assessment of what descriptor fits the work done.

Third, after the user confirms or modifies the path and filename, write the development notes following the required structure and content guidelines described below.

## Required Structure

The development notes must include the following sections, written in English.

The Background section should explain why this work was initiated. What problem was the user trying to solve? What was the original question or request? This section provides the motivational context that helps the next AI understand not just what was done but why it was needed.

The Key Decisions section should document important choices made during the conversation, especially those where the user explicitly stated a preference. Include the user's original words in their original language when they made significant decisions. For example, if the user said "沒有 user 吼" to reject a proposed architecture, quote that exact phrase and explain what it meant in context. These user quotes are crucial because they represent explicit requirements that must be respected in future work.

The Architecture section should describe the technical structure of what was built. This includes directory layouts, file purposes, class hierarchies, data schemas, and how components interact. Use code blocks for schemas, directory trees, and other structured information where appropriate. The goal is to give the next AI a complete mental model of the system.

The Implementation Progress section should chronicle what was actually done, in roughly chronological order. Describe each phase of work, what was attempted, what problems were encountered, and how they were resolved. This narrative helps the next AI understand not just the final state but the journey that led there, which is valuable context for understanding design decisions.

The Current State section should summarize what is working, what has been tested, and what the results were. Include specific metrics, test outputs, or other concrete evidence of the current functionality. This gives the next AI a clear starting point.

The Next Steps section should list features or tasks that were discussed but not yet implemented, or that logically follow from the current work. Be specific about what remains to be done so the next AI can continue productively.

Finally, include a Technical Notes for Next AI section with practical information like how to run tests, common commands, known quirks or gotchas, and any other operational knowledge that would help the next AI work effectively on this project.

## Writing Style Guidelines

Write in complete paragraphs with full sentences. Do not use bullet points for the main content, as paragraph form provides better context and flow for AI comprehension. However, you may use structured formats like code blocks, tables, or lists for genuinely structured data such as directory trees, configuration schemas, test results, or command references.

Be thorough and explicit. Information that seems obvious may not be obvious to an AI encountering this project for the first time. Err on the side of providing more context rather than less.

Include specific file paths, function names, class names, and other concrete references. Vague descriptions are less useful than precise pointers to actual code.

When describing user decisions, always include the rationale if it was stated or can be inferred. Understanding why a decision was made is often more valuable than knowing what the decision was.

## After Writing the Notes

Once you have written the development notes, inform the user that the notes have been saved and provide a brief summary of what was documented. Remind them that future AI assistants can read this file to quickly get up to speed on the project.
