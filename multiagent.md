# AI Software Development Team --  Flow Overview

## Project Goal

Build an AI-powered multi-agent software development system that can
take a user requirement and automatically design, code, test, and
prepare a full-stack application.

------------------------------------------------------------------------

## Step-by-Step Normal Flow

### 1. User Requirement

The user provides a project idea. Example: "Build a task management app
with login and dashboard."

------------------------------------------------------------------------

### 2. PM Agent -- Requirement Understanding

-   Analyzes the user request
-   Asks clarifying questions if needed
-   Converts the request into a clear structured specification

If clarification is needed → ask user\
If clear → move forward

------------------------------------------------------------------------

### 3. Architect Agent -- System Design

Designs: - Database schema - API endpoints - Frontend pages - Folder
structure - Technology stack

Creates a complete blueprint of the system.

------------------------------------------------------------------------

### 4. Blueprint Validation

System checks: - Every API connects to the database properly - Every
frontend page connects to valid APIs - No missing entities - No broken
relationships

If invalid → fix architecture\
If valid → continue

------------------------------------------------------------------------

### 5. Planner Agent -- Task Breakdown

Divides project into phases:

Phase 1 -- Project Setup & Database\
Phase 2 -- Backend APIs\
Phase 3 -- Frontend\
Phase 4 -- Integration

Each phase contains smaller development tasks.

------------------------------------------------------------------------

### 6. Sandbox Setup

System: - Creates Docker container - Initializes project structure -
Installs dependencies - Connects database - Initializes Git repository

Now the coding environment is ready.

------------------------------------------------------------------------

### 7. Development Loop (Main Coding Process)

For each task:

A. Context Builder prepares required information\
B. Coder Agent writes the code\
C. Reviewer Agent checks for bugs and issues

If rejected → fix and retry\
If approved → continue

------------------------------------------------------------------------

### 8. Code Execution & Testing

Executor Agent runs the code inside Docker.

If execution passes: - Create Git snapshot (save stable version)

If execution fails: - Debugger Agent analyzes errors - Fixes the issue -
Can rollback to last working version if necessary

------------------------------------------------------------------------

### 9. Repeat for All Tasks

System continues:

Select Task → Code → Review → Execute → Save Snapshot

Until all tasks are completed.

------------------------------------------------------------------------

### 10. Present Project to User

System: - Runs the full application - Provides working preview - Shows
token usage and cost summary

User can test the application.

------------------------------------------------------------------------

### 11. Feedback Loop

User can: - Report bugs - Request changes - Add new features

System converts feedback into new tasks and repeats development loop.

------------------------------------------------------------------------

### 12. Deployment

System generates: - Deployment configuration files - Hosting
instructions - Final clean project version

Project is ready for deployment.

------------------------------------------------------------------------

## Final Summary

User → Understand → Design → Plan → Setup → Code → Review → Test → Fix →
Deliver → Deploy

This is the normal structured workflow of the AI-powered multi-agent
development system.
