# Contributing to Open Pace

Thank you for your interest in contributing to Open Pace! This document outlines how to contribute effectively to this project.

## 🤖 AI Contributions Welcome

**AI-generated contributions are explicitly allowed and encouraged.** This project recognizes that AI tools can be valuable collaborators in software development. Contributions from AI assistants (such as GitHub Copilot, ChatGPT, Claude, or similar tools) are treated the same as human contributions.

### AI Contribution Guidelines

- ✅ **AI-generated code is welcome** - Feel free to use AI tools to help write code, documentation, or tests
- ✅ **AI-assisted reasoning is encouraged** - Use AI to help think through problems, design solutions, or review code
- ✅ **Attribution is optional** - You don't need to disclose AI assistance, but you may if you wish. Make sure you comply with the Apache 2 license.
- ✅ **Quality standards apply** - AI-generated contributions must still meet the same quality and testing standards as human contributions
- ✅ **Maintainer review required** - All contributions, whether AI-assisted or not, are subject to maintainer review and approval

### High-Fidelity Thinking Models

When contributing (with or without AI assistance), we encourage the use of **high-fidelity thinking models** - structured reasoning processes that lead to well-thought-out contributions. These models help ensure contributions are:

- **Well-reasoned**: Clear rationale for design decisions
- **Thorough**: Consideration of edge cases, alternatives, and trade-offs
- **Documented**: Explanation of why choices were made
- **Consistent**: Alignment with existing patterns and architecture

#### Recommended Thinking Models

**1. Problem-Solution Analysis**
```
1. Clearly define the problem or need
2. Identify constraints and requirements
3. Explore multiple solution approaches
4. Evaluate trade-offs (performance, complexity, maintainability)
5. Select the best approach with rationale
6. Document the decision
```

**2. Architecture-First Thinking**
```
1. Understand existing architecture and patterns
2. Identify where the contribution fits
3. Consider impact on existing systems
4. Ensure consistency with established patterns
5. Document architectural decisions
```

**3. Test-Driven Reasoning**
```
1. Define expected behavior and outcomes
2. Consider edge cases and error conditions
3. Plan test coverage before implementation
4. Implement to satisfy tests
5. Verify integration with existing tests
```

**4. Documentation-Driven Development**
```
1. Document the "why" before the "what"
2. Explain ActivityPub concepts and patterns
3. Provide examples and use cases
4. Include troubleshooting guidance
5. Update relevant strategy documents
```

**5. Incremental Refinement**
```
1. Start with minimal working solution
2. Iterate based on feedback and testing
3. Refine implementation incrementally
4. Maintain working state at each step
5. Document evolution and lessons learned
```

#### Applying Thinking Models

When submitting contributions, consider:

- **Have you thought through the problem?** - Use problem-solution analysis
- **Does it fit the architecture?** - Apply architecture-first thinking
- **Is it well-tested?** - Follow test-driven reasoning
- **Is it documented?** - Use documentation-driven development
- **Is it incremental?** - Apply incremental refinement

You don't need to explicitly document which thinking model you used, but contributions that demonstrate clear reasoning and thorough consideration are more likely to be accepted.

## 📋 Contribution Process

### Before You Start

1. **Read the documentation**:
   - [Implementation Strategy](IMPLEMENTATION_STRATEGY.md) - Overall approach and principles
   - [Consistency Checklist](CONSISTENCY_CHECKLIST.md) - Quality standards
   - **[Quarkus Tech Stack](docs/QUARKUS_TECH_STACK.md)** - **Technology choices and dependency philosophy** ⚠️ **Important: Read this before suggesting technology changes**
   - Relevant strategy documents in `docs/` folder

2. **Check existing issues**: Look for open issues or discussions that might relate to your contribution

3. **Understand the project structure**: Single codebase that evolves incrementally. Use git tags to access previous sprint states (e.g., `git checkout v0.1.0-sprint1`)

### Making Contributions

#### 1. Fork and Clone

```bash
# Fork the repository on GitHub, then:
git clone https://github.com/your-username/open-pace.git
cd open-pace
```

#### 2. Set Up Development Environment

- Java 21+ (LTS)
- Maven 3.8+
- Podman (for Quarkus Dev Services)
- Your preferred IDE

#### 3. Create a Feature Branch

Follow the git workflow from [Implementation Strategy](IMPLEMENTATION_STRATEGY.md):

```bash
git checkout develop
git pull origin develop
git checkout -b feature/your-feature-name
```

**Branch naming conventions**:
- `feature/sprint-X` - New sprint implementation
- `feature/description` - New feature (e.g., `feature/gpx-parsing`)
- `fix/description` - Bug fix (e.g., `fix/webfinger-discovery`)
- `docs/description` - Documentation updates
- `refactor/description` - Code refactoring

#### 4. Make Your Changes

- **Follow existing patterns**: Look at previous sprints for consistency
- **Write tests**: Include unit and integration tests
- **Update documentation**: Update relevant docs in `docs/` folder
- **Follow code style**: Use consistent formatting and naming
- **Commit frequently**: Use clear, conventional commit messages

#### 5. Commit Messages

Follow [Conventional Commits](https://www.conventionalcommits.org/) format:

```
<type>(<scope>): <subject>

<body>

<footer>
```

**Types**:
- `feat`: New feature
- `fix`: Bug fix
- `docs`: Documentation
- `refactor`: Code refactoring
- `test`: Tests
- `chore`: Maintenance

**Examples**:
```
feat(sprint1): implement WebFinger discovery endpoint

Adds /.well-known/webfinger endpoint for ActivityPub
actor discovery. Supports resource parameter query.

Closes #123
```

```
fix(sprint2): correct JSON-LD context for RunActivity

Fixes incorrect @context URL in RunActivity serialization.
Now uses correct FediRun namespace.

Fixes #456
```

#### 6. Test Your Changes

```bash
# Run tests
./mvnw test

# Test federation (if applicable)
# Follow manual testing steps from sprint documentation
```

#### 7. Update Documentation

- Update relevant strategy documents if you've established new patterns
- Update sprint-specific README if applicable
- Add examples and use cases
- Document any new concepts or decisions

#### 8. Push and Create Pull Request

```bash
git push origin feature/your-feature-name
```

Then create a Pull Request on GitHub:

- **Target branch**: `develop` (for new features) or `main` (for hotfixes)
- **Title**: Clear, descriptive title
- **Description**: 
  - What changes were made
  - Why the changes were made
  - How to test the changes
  - Any breaking changes
  - Related issues (if any)

## 🎯 What to Contribute

### Welcome Contributions

- ✅ **Bug fixes**: Fix issues in existing sprints
- ✅ **Feature enhancements**: Add features to existing sprints
- ✅ **New sprint implementations**: Implement planned sprints
- ✅ **Documentation improvements**: Clarify, expand, or fix documentation
- ✅ **Test coverage**: Add missing tests
- ✅ **Code quality**: Refactoring for clarity or performance
- ✅ **Examples**: Add working examples or tutorials
- ✅ **ActivityPub improvements**: Better federation, compliance, or interop

### Contribution Priorities

1. **Completing planned sprints** (Sprints 1-10)
2. **Bug fixes** for existing functionality
3. **Documentation** improvements
4. **Test coverage** improvements
5. **New features** that align with project goals

### What Not to Contribute (Yet)

- ❌ Features that don't align with the sprint structure
- ❌ Breaking changes without discussion
- ❌ Dependencies that conflict with the tech stack
- ❌ Features that require significant architectural changes without prior discussion

## 🔍 Review Process

### Maintainer Decision Authority

**All decisions about inclusion are made solely by the maintainer.** This includes:

- Whether to accept or reject contributions
- What changes are requested before acceptance
- Priority of contributions
- Architectural decisions
- Feature inclusion or exclusion

### Review Criteria

Contributions are evaluated based on:

1. **Quality**: Code quality, test coverage, documentation
2. **Consistency**: Alignment with existing patterns and architecture
3. **Completeness**: Working, tested, and documented
4. **Relevance**: Fits project goals and sprint structure
5. **Reasoning**: Clear rationale and consideration of alternatives

### Review Process

1. **Initial review**: Maintainer reviews the PR
2. **Feedback**: Maintainer may request changes or ask questions
3. **Iteration**: Contributor addresses feedback
4. **Final decision**: Maintainer approves or closes the PR
5. **Merge**: Approved PRs are merged to `develop` or `main`

### Response Time

- The maintainer will review PRs as time permits
- Complex contributions may take longer to review
- Feel free to ping if a PR has been open for a while without feedback

## 📐 Code Standards

### Code Style

- Follow Java conventions
- Use consistent formatting (consider using formatter configuration)
- Write self-documenting code with clear variable names
- Add comments for ActivityPub-specific concepts
- Keep methods focused and reasonably sized

### Testing Standards

- **Unit tests**: Test business logic and utilities
- **Integration tests**: Test ActivityPub endpoints and federation
- **Manual tests**: Include test scripts or instructions
- **Coverage**: Aim for good test coverage, especially for critical paths

### Documentation Standards

- **Code comments**: Explain "why", not just "what"
- **ActivityPub concepts**: Document ActivityPub-specific patterns
- **Examples**: Include working examples
- **Troubleshooting**: Document common issues and solutions

### Architecture Consistency

- Follow patterns established in previous sprints
- Use existing package structure
- Follow API design patterns from [API Design](docs/API_DESIGN.md)
- Follow error handling patterns from [Error Handling Strategy](docs/ERROR_HANDLING_STRATEGY.md)
- Document new patterns in relevant strategy documents

### Technology Stack

**Important**: This project uses **Quarkus** as its core framework. Before suggesting alternative technologies or frameworks, please read the [Quarkus Tech Stack](docs/QUARKUS_TECH_STACK.md) document, which explains:

- Why Quarkus was chosen
- The dependency philosophy (focus on Quarkus extensions)
- That core technology choices (Quarkus, Java, PostgreSQL) are established and final

We appreciate contributions that work within the established technology stack. Suggestions for alternative frameworks or languages will not be considered, as these decisions allow us to focus on building great ActivityPub functionality.

### Apache 2 License Headers

**All source code files must include the Apache 2.0 license header.** This is mandatory for compliance with the Apache 2.0 license under which this project is distributed.

#### Required Files

The following file types **must** include the license header:

- ✅ **Java source files** (`.java`)
- ✅ **Java test files** (`.java` in test directories)
- ✅ **Configuration files** that are part of the source (if applicable)
- ❌ **Generated files** (excluded - e.g., generated by annotation processors)
- ❌ **Third-party code** (excluded - maintain their original headers)

#### License Header Format

**For Java files**, place the license header at the very top of the file, before the `package` statement:

```java
/*
 * Copyright [yyyy] [name of copyright owner]
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *     http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
package org.openpace.core;
```

#### Copyright Notice

Replace the placeholders in the copyright notice:

- `[yyyy]` - Replace with the current year (e.g., `2024`)
- `[name of copyright owner]` - Replace with the copyright owner name (typically the project name or your name if contributing)

**Example for Open Pace project**:

```java
/*
 * Copyright 2024 Open Pace Contributors
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * ...
 */
```

Or if you're contributing as an individual:

```java
/*
 * Copyright 2024 [Your Name]
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * ...
 */
```

#### File Structure

The license header must be:

1. **At the very top** of the file (first lines)
2. **Before the `package` statement** (for Java files)
3. **In a comment block** using `/* */` style comments
4. **Exactly formatted** as shown above (preserve spacing and line breaks)

#### Examples

**Correct - Java class file**:
```java
/*
 * Copyright 2024 Open Pace Contributors
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *     http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
package org.openpace.core;

import jakarta.ws.rs.GET;
import jakarta.ws.rs.Path;

@Path("/api/health")
public class HealthResource {
    // ...
}
```

**Correct - Test file**:
```java
/*
 * Copyright 2024 Open Pace Contributors
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *     http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
package org.openpace.core;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class HealthResourceTest {
    // ...
}
```

#### Verification

Before submitting a PR, verify that:

- [ ] All new `.java` files include the license header
- [ ] All modified `.java` files retain their license headers
- [ ] Copyright year is current (or appropriate for the file)
- [ ] Header format matches the template exactly
- [ ] Header is placed before the `package` statement

#### Automated Checking

You can use tools to verify license headers:

- **License Maven Plugin**: Automatically add/verify license headers
- **Checkstyle**: Configure to check for license headers
- **Manual review**: Check files before committing

#### AI-Generated Code

**Important**: When using AI tools to generate code, ensure the generated code includes the Apache 2.0 license header. AI tools may not automatically include license headers, so you must add them manually.

**Compliance**: As stated in the AI Contribution Guidelines, you must ensure AI-generated contributions comply with the Apache 2.0 license requirements, including proper license header inclusion.

#### Questions?

If you're unsure about:
- Whether a file needs a license header
- The correct copyright owner name
- How to format the header for a specific file type

Please ask in your PR or open an issue for clarification.

## 🐛 Reporting Issues

### Bug Reports

When reporting bugs, please include:

1. **Description**: Clear description of the issue
2. **Steps to reproduce**: Detailed steps to reproduce the bug
3. **Expected behavior**: What should happen
4. **Actual behavior**: What actually happens
5. **Environment**: Java version, OS, relevant versions
6. **Sprint**: Which sprint/version contains the issue (e.g., `v0.1.0-sprint1` or current `develop`)
7. **Logs**: Relevant error messages or stack traces

### Feature Requests

For feature requests:

1. **Description**: Clear description of the feature
2. **Use case**: Why this feature is needed
3. **Sprint alignment**: Which sprint it fits into (or if it's a new sprint)
4. **Alternatives**: Other approaches you've considered

## 💬 Communication

### Discussion Channels

- **GitHub Issues**: For bugs, feature requests, and discussions
- **Pull Requests**: For code review discussions
- **GitHub Discussions**: For general questions and community discussion (if enabled)

### Code of Conduct

- Be respectful and constructive
- Focus on the code, not the person
- Provide helpful feedback
- Accept feedback gracefully
- Remember that maintainer decisions are final

## 📚 Resources

### Learning Resources

- [ActivityPub Specification](https://www.w3.org/TR/activitypub/)
- [Quarkus Documentation](https://quarkus.io/)
- [Vert.x Documentation](https://vertx.io/)
- [Implementation Strategy](IMPLEMENTATION_STRATEGY.md)

### Project Documentation

- [Implementation Strategy](IMPLEMENTATION_STRATEGY.md) - Overall approach
- [Consistency Checklist](CONSISTENCY_CHECKLIST.md) - Quality standards
- [Database Design](docs/DATABASE_DESIGN.md) - Schema patterns
- [API Design](docs/API_DESIGN.md) - Endpoint organization
- [Error Handling Strategy](docs/ERROR_HANDLING_STRATEGY.md) - Error patterns

## ✅ Checklist Before Submitting

Before submitting a PR, ensure:

- [ ] **License headers included** - All source code files have Apache 2.0 license headers
- [ ] Code follows existing patterns and style
- [ ] Tests are written and passing
- [ ] Documentation is updated
- [ ] Commit messages follow conventional commits
- [ ] Branch is up to date with `develop`
- [ ] PR description is clear and complete
- [ ] No breaking changes (or breaking changes are documented)
- [ ] Federation tested (if applicable)
- [ ] Relevant strategy documents updated (if new patterns established)

## 🙏 Thank You

Thank you for contributing to Open Pace! Every contribution, whether code, documentation, bug reports, or feedback, helps make this project better. We appreciate your time and effort.

---

**Remember**: All contributions are subject to maintainer review and approval. The maintainer makes all final decisions about inclusion, but we value and appreciate every contribution that helps move the project forward.
