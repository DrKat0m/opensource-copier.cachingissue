# Contribution [#]: [Copier Template Caching Issue]

**Contribution Number:** 1  
**Student:** Kartavya Suhagiya  
**Issue:** [[GitHub issue link](https://github.com/copier-org/copier/issues/450)]  
**Status:** [Phase I] [In Progress]

---

## Why I Chose This Issue

This issue interests me because it focuses on improving the performance and developer experience of Copier. The current implementation downloads or fetches remote Git templates every time they are used, which can introduce noticeable delays, especially when repeatedly generating or updating projects from the same template repository. Performance optimizations that have a direct impact on users are particularly interesting to me because they combine software design, Git internals, and practical engineering considerations.

From a learning perspective, this issue provides an opportunity to gain a deeper understanding of how Copier interacts with Git repositories, how repository caching strategies are implemented, and how features such as Git mirrors and worktrees can be used to improve efficiency. I also hope to learn more about contributing to a mature open-source codebase, including understanding existing architecture, handling edge cases, and collaborating with maintainers during the review process.

---

## Understanding the Issue

### Problem Description

When Copier uses a template stored in a remote Git repository, it performs cloning or fetching operations that can take several seconds, even when the same template has been used previously. This repeated network activity makes project generation and updates slower than necessary.

The issue proposes adding a local cache for Git templates so that Copier can reuse previously downloaded repository data instead of downloading it again for every operation.

### Expected Behavior

Copier should maintain a local cache of remote Git templates. When a template repository is requested, Copier should first check the cache and reuse the existing repository data whenever possible. The cache should be updated using Git fetch operations when needed, and temporary working copies (such as Git worktrees) can be created from the cached repository for project generation or updates.

This would significantly reduce the amount of network traffic and improve the speed of repeated operations involving the same template repository.

### Current Behavior

Currently, Copier performs Git operations directly against the remote repository each time a template is used. Although previous optimizations such as blob-less cloning have improved performance, users still report that operations involving remote templates are considerably slower than using locally available templates.

For example, a user reported that updating from a remote template took approximately 5.5 seconds, while using a locally checked-out template took approximately 0.7 seconds. The absence of a persistent local cache means Copier cannot fully benefit from data that has already been downloaded.

### Affected Components

The primary components involved are:

The Git template retrieval and cloning logic used by Copier.
Code responsible for resolving and preparing template repositories before project generation or updates.
Temporary workspace management used during template processing.
Potential new cache management functionality, including:
Creating and maintaining cached Git mirrors.
Updating cached repositories through fetch operations.
Creating temporary worktrees from cached repositories.
Handling cache cleanup and repository synchronization.

Additional consideration may be required for repository references such as branches, tags, and commits, as well as concurrent access to the cache by multiple Copier processes.

---

## Reproduction Process

### Environment Setup

[Notes on setting up your local development environment - challenges you faced, how you solved them]

### Steps to Reproduce

1. [Step 1]
2. [Step 2]
3. [Observed result]

### Reproduction Evidence

- **Commit showing reproduction:** [Link to commit in your fork]
- **Screenshots/logs:** [If applicable]
- **My findings:** [What you discovered during reproduction]

---

## Solution Approach

### Analysis

[Your analysis of the root cause - what's causing the issue?]

### Proposed Solution

[High-level description of your fix approach]

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** [Restate the problem]

**Match:** [What similar patterns/solutions exist in the codebase?]

**Plan:** [Step-by-step implementation plan]
1. [Modify file X to do Y]
2. [Add function Z]
3. [Update tests]

**Implement:** [Link to your branch/commits as you work]

**Review:** [Self-review checklist - does it follow the project's contribution guidelines?]

**Evaluate:** [How will you verify it works?]

---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: [Description]
- [ ] Test case 2: [Description]
- [ ] Test case 3: [Description]

### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

### Manual Testing

[What you tested manually and results]

---

## Implementation Notes

### Week [X] Progress

[What you built this week, challenges faced, decisions made]

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
