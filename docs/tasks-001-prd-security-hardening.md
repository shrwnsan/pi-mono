# Tasks-001-PRD-Security-Hardening: Implementation Task Breakdown

**PRD Reference**: PRD-001-security-hardening.md
**Created**: 2025-11-13
**Updated**: 2025-11-14 (Security Assessment Integration)
**Team**: Security Development Team
**Status**: Ready for Assignment - **URGENT**

## Overview

This document breaks down the security hardening initiative into manageable tasks that can be worked on in parallel by junior developers. Each task includes detailed requirements, acceptance criteria, and implementation guidance.

## **Security Assessment Summary (November 14, 2025)**
**Production Status**: **NOT READY - 7 CRITICAL vulnerabilities confirmed**

**Immediate Threats Identified**:
- AI can read/write any system file (`/etc/passwd`, `~/.ssh/id_rsa`, API keys)
- AI can execute any shell command (`rm -rf /`, malware installation)
- API keys stored in browser with `dangerouslyAllowBrowser: true`
- SSH operations without host key verification

**Code Review References**: All task specifications include exact file paths and line numbers from security assessment.

## Phase 1: Critical Security Foundation (Weeks 1-2)

### Task 1.1: File System Sandboxing Implementation
**Priority**: Critical
**Estimated Time**: 3-4 days
**Assignee**: Junior Developer 1
**Dependencies**: None

**Description**: Implement sandboxed file operations to prevent unrestricted filesystem access.

**Requirements**:
- Create sandboxed filesystem interface in `packages/coding-agent/src/tools/filesystem-sandbox.ts`
- Modify existing Read, Write, and Edit tools to use sandbox
- Implement path traversal protection
- Add file permission validation
- Create safe working directory structure

**Implementation Details**:
```typescript
// Example sandbox interface
interface FilesystemSandbox {
  allowedPaths: string[];
  maxSizeBytes: number;
  maxFiles: number;
  blockedPatterns: RegExp[];
}

// Methods to implement
- validatePath(path: string): boolean
- canonicalizePath(path: string): string
- checkPermissions(path: string, operation: 'read' | 'write' | 'execute'): boolean
- enforceQuotas(): boolean
```

**Files to Modify** (Specific Lines from Security Assessment):
- `packages/coding-agent/src/tools/read.ts`
  - **Line 60**: Replace `const absolutePath = resolvePath(expandPath(path));`
  - **Lines 86-87**: Replace unrestricted `await access(absolutePath, constants.R_OK);`
- `packages/coding-agent/src/tools/write.ts`
  - **Line 32**: Replace unrestricted path resolution
  - **Lines 58, 66**: Replace `await writeFile(absolutePath, content, "utf-8");`
- `packages/coding-agent/src/tools/edit.ts`
  - **Lines 132, 161, 176, 218**: Replace unrestricted file modification operations
- `packages/coding-agent/src/tools/bash.ts` (remove file operation capabilities)

**Current Vulnerable Code Patterns**:
```typescript
// VULNERABLE: Direct file system access
const absolutePath = resolvePath(expandPath(path));  // NO VALIDATION
await access(absolutePath, constants.R_OK);          // UNRESTRICTED
await writeFile(absolutePath, content, "utf-8");    // NO SANDBOX
```

**Acceptance Criteria**:
- [ ] All file operations restricted to sandbox directory
- [ ] Path traversal attacks (../../../etc/passwd) blocked
- [ ] File size and count quotas enforced
- [ ] Proper error handling for blocked operations
- [ ] Unit tests with 90%+ coverage
- [ ] Integration tests with security test cases

**Testing Checklist**:
- Path traversal injection attempts
- Symbolic link attacks
- File permission bypass attempts
- Quota enforcement testing
- Performance benchmarking

---

### Task 1.2: Command Execution Security Hardening
**Priority**: Critical
**Estimated Time**: 4-5 days
**Assignee**: Junior Developer 2
**Dependencies**: None

**Description**: Secure shell command execution with proper filtering and isolation.

**Requirements**:
- Implement command whitelist system
- Add argument validation and sanitization
- Create isolated execution environment
- Add execution timeouts and resource limits
- Implement comprehensive audit logging

**Implementation Details**:
```typescript
// Command whitelist configuration
interface CommandWhitelist {
  allowedCommands: string[];
  blockedPatterns: RegExp[];
  timeoutMs: number;
  maxMemoryMB: number;
  maxProcesses: number;
}

// Methods to implement
- validateCommand(command: string): ValidationResult
- sanitizeArguments(args: string[]): string[]
- createIsolatedEnvironment(): ExecutionEnvironment
- enforceResourceLimits(process: ChildProcess): void
```

**Files to Modify** (Specific Lines from Security Assessment):
- `packages/coding-agent/src/tools/bash.ts`
  - **Lines 82-84**: Replace direct `spawn()` calls with validation
  - **Current vulnerable code**:
    ```typescript
    const child = spawn(shell, [...args, command], {
        detached: true,
        stdio: ["ignore", "pipe", "pipe"],
    });
    ```
- Create: `packages/coding-agent/src/security/command-validator.ts`
- Create: `packages/coding-agent/src/security/execution-sandbox.ts`

**Critical Issues to Address**:
- AI can execute ANY shell command including `rm -rf /`, `sudo su`
- No command whitelisting or validation
- No resource limits or timeouts
- Unrestricted network access
- No audit logging of executed commands

**Acceptance Criteria**:
- [ ] Only whitelisted commands can execute
- [ ] Dangerous command patterns blocked
- [ ] Resource limits enforced (CPU, memory, time)
- [ ] All executions logged with full details
- [ ] Process cleanup and isolation working
- [ ] Comprehensive test coverage

**Security Test Cases**:
- Command injection attempts
- Argument pollution attacks
- Resource exhaustion testing
- Process escape attempts
- Timeout enforcement verification

---

### Task 1.3: Basic Input Validation Framework
**Priority**: High
**Estimated Time**: 2-3 days
**Assignee**: Junior Developer 3
**Dependencies**: None

**Description**: Create comprehensive input validation framework to prevent injection attacks.

**Requirements**:
- Create input sanitization utilities
- Implement validation schemas for all tool inputs
- Add content length limits
- Create pattern-based detection for malicious content

**Implementation Details**:
```typescript
// Input validation interface
interface InputValidator {
  validateString(input: string, rules: ValidationRules): ValidationResult;
  sanitize(input: string): string;
  detectMaliciousPatterns(input: string): boolean;
}

// Validation rules
interface ValidationRules {
  maxLength: number;
  allowedPatterns: RegExp[];
  blockedPatterns: RegExp[];
  encoding: 'utf-8' | 'ascii';
}
```

**Files to Create**:
- `packages/coding-agent/src/security/input-validator.ts`
- `packages/coding-agent/src/security/patterns.ts` (malicious pattern definitions)

**Files to Modify**:
- All tool files in `packages/coding-agent/src/tools/`

**Acceptance Criteria**:
- [ ] All tool inputs validated before processing
- [ ] Malicious patterns detected and blocked
- [ ] Input sanitization working correctly
- [ ] Proper error messages for invalid inputs
- [ ] Performance impact minimal (<5ms per validation)
- [ ] Comprehensive test coverage

**Test Cases**:
- SQL injection patterns
- XSS attempts
- Command injection strings
- Path traversal attempts
- Buffer overflow attempts

---

## Phase 2: API Security Foundation (Weeks 3-4)

### Task 2.1: Server-Side API Proxy Implementation
**Priority**: High
**Estimated Time**: 5-6 days
**Assignee**: Junior Developer 1
**Dependencies**: Task 1.1, 1.2, 1.3

**Description**: Create server-side API proxy to eliminate client-side API key storage using environment variable approach.

**Requirements**:
- Build API proxy server for external API calls
- **Implement environment variable-based key management**
- Add request validation and rate limiting
- Remove all client-side key storage mechanisms

**Implementation Details**:
```typescript
// Environment variable-based approach
class EnvironmentKeyManager {
  // Load keys from secure environment variables
  private anthropicKey = process.env.ANTHROPIC_API_KEY;
  private openaiKey = process.env.OPENAI_API_KEY;
  private geminiKey = process.env.GEMINI_API_KEY;

  // No client-side key storage - server only
  getProviderKey(provider: string): string | null {
    switch(provider) {
      case 'anthropic': return this.anthropicKey;
      case 'openai': return this.openaiKey;
      case 'gemini': return this.geminiKey;
      default: return null;
    }
  }
}

// API proxy interface
interface APIProxy {
  proxyRequest(endpoint: string, provider: string, request: any): Promise<any>;
  // No apiKey parameter - keys loaded from environment
  generateShortLivedToken(userId: string): string;
  validateToken(token: string): boolean;
  enforceRateLimit(userId: string): boolean;
}
```

**Files to Create**:
- `packages/api-proxy/src/server.ts`
- `packages/api-proxy/src/environment-key-manager.ts` (NEW - env var approach)
- `packages/api-proxy/src/token-manager.ts`
- `packages/api-proxy/src/rate-limiter.ts`
- `.env.example` (template for required environment variables)

**Files to Modify** (Specific Lines from Security Assessment):
- `packages/web-ui/src/storage/stores/provider-keys-store.ts` (**DELETE or replace with proxy client**)
  - **Lines 18-19**: Remove client-side API key storage entirely
  - **Current vulnerable code**: `await this.getBackend().set("provider-keys", provider, key);`
- `packages/ai/src/providers/anthropic.ts`
  - **Lines 115, 285-322**: Remove `dangerouslyAllowBrowser: true`
  - **Remove all client-side API key handling**
- `packages/agent/src/api-clients/` (update to use server-side proxy)

**Critical Security Issues to Address**:
- ❌ **API keys stored in browser IndexedDB/localStorage** → ✅ **Environment variables on server**
- ❌ **Keys accessible to malicious browser extensions** → ✅ **Zero client exposure**
- ❌ **dangerouslyAllowBrowser: true** → ✅ **Server-side proxy only**
- ❌ **No server-side proxy implementation** → ✅ **Full proxy implementation**
- ❌ **Keys transmitted in plain text to client** → ✅ **Keys never reach client**

**Environment Variable Setup**:
```bash
# Required environment variables
ANTHROPIC_API_KEY=sk-ant-...
OPENAI_API_KEY=sk-...
GEMINI_API_KEY=AIza...

# Security settings
API_PROXY_PORT=3001
JWT_SECRET=your-jwt-secret
RATE_LIMIT_REQUESTS=100
RATE_LIMIT_WINDOW=900000
```

**Acceptance Criteria**:
- [ ] **API keys loaded ONLY from environment variables**
- [ ] **Zero client-side key storage** (delete provider-keys-store.ts)
- [ ] **Remove dangerouslyAllowBrowser: true from all providers**
- [ ] **Server-side proxy handles all external API calls**
- [ ] **Rate limiting enforced per user/token**
- [ ] **All API calls logged with key identification**
- [ ] **Environment variable validation on startup**
- [ ] **Fallback handling for missing environment variables**

---

### Task 2.2: SSH Security Hardening
**Priority**: High
**Estimated Time**: 4-5 days
**Assignee**: Junior Developer 2
**Dependencies**: Task 1.2, 1.3

**Description**: Secure SSH and remote execution capabilities.

**Requirements**:
- Implement SSH host key verification
- Add command injection protection for SSH
- Create secure file transfer validation
- Add session isolation and monitoring

**Implementation Details**:
```typescript
// SSH security interface
interface SSHSecurity {
  verifyHostKey(host: string, key: string): boolean;
  validateSSHCommand(command: string): ValidationResult;
  secureFileTransfer(localPath: string, remotePath: string): Promise<boolean>;
  monitorSSHSessions(): SessionMonitor;
}
```

**Files to Modify** (Specific Lines from Security Assessment):
- `packages/pods/src/ssh.ts`
  - **Lines 12-32**: Replace `const sshParts = sshCmd.split(" ").filter((p) => p);`
  - **Lines 68-98**: Replace unrestricted `spawn(sshBinary, sshArgs, { stdio: ["ignore", "pipe", "pipe"] });`
  - **Current vulnerable code**: No host key verification, command injection possible
- Create: `packages/pods/src/ssh-security.ts`
- Create: `packages/pods/src/host-key-verifier.ts`

**Critical Security Issues**:
- SSH commands executed without host key verification (man-in-the-middle possible)
- Command injection through SSH command parsing
- No session isolation or monitoring
- Unrestricted file transfer capabilities
- No audit trail of SSH operations

**Acceptance Criteria**:
- [ ] SSH host key verification mandatory
- [ ] Command injection protection working
- [ ] File transfers validated for malicious content
- [ ] Session monitoring and logging functional
- [ ] Proper cleanup of SSH connections
- [ ] Security test coverage

---

## Phase 3: Advanced Security Features (Weeks 5-6)

### Task 3.1: Content Scanning Implementation
**Priority**: Medium
**Estimated Time**: 4-5 days
**Assignee**: Junior Developer 3
**Dependencies**: Task 1.3, 2.1

**Description**: Implement content scanning for malicious code patterns.

**Requirements**:
- Create malware and vulnerability scanner
- Implement static code analysis
- Add pattern matching for security issues
- Create quarantine system for suspicious content

**Files to Create**:
- `packages/security/src/content-scanner.ts`
- `packages/security/src/pattern-matcher.ts`
- `packages/security/src/quarantine.ts`

**Acceptance Criteria**:
- [ ] Malicious code patterns detected
- [ ] Known vulnerabilities flagged
- [ ] Suspicious content quarantined
- [ ] False positive rate <5%
- [ ] Performance impact minimal

---

### Task 3.2: Security Monitoring and Logging
**Priority**: Medium
**Estimated Time**: 3-4 days
**Assignee**: Junior Developer 1
**Dependencies**: All previous tasks

**Description**: Implement comprehensive security monitoring and audit logging.

**Requirements**:
- Create security event logging system
- Implement real-time monitoring
- Add alerting for security events
- Create security dashboard

**Files to Create**:
- `packages/security/src/audit-logger.ts`
- `packages/security/src/event-monitor.ts`
- `packages/security/src/alerting.ts`
- `packages/web-ui/src/components/security-dashboard/`

**Acceptance Criteria**:
- [ ] All security events logged
- [ ] Real-time monitoring working
- [ ] Alerting system functional
- [ ] Security dashboard displays relevant metrics
- [ ] Log retention and rotation implemented

---

## Testing Strategy

### Unit Testing Requirements
- All new code must have 90%+ test coverage
- Security-specific test cases for each feature
- Mock external dependencies for isolated testing
- Performance benchmarks for all security features

### Integration Testing
- End-to-end security workflow testing
- Penetration testing scenarios
- Load testing with security features enabled
- Cross-package integration testing

### Security Testing
- OWASP Top 10 vulnerability testing
- Penetration testing by security team
- Static code analysis security scanning
- Dependency vulnerability scanning

## Development Guidelines

### Code Review Process
- All code must be reviewed by senior developer
- Security-focused code review checklist
- Automated security scanning in CI/CD
- Documentation requirements for all security features

### Documentation Requirements
- API documentation for all security interfaces
- Security architecture diagrams
- Incident response procedures
- User security guidelines

### Performance Requirements
- Security features must not degrade performance >10%
- Memory usage increases <20%
- Response time increases <50ms
- Resource utilization monitoring

## Deployment Strategy

### Staging Environment
- Security features tested in staging first
- Performance benchmarking in staging
- Security scanning in staging
- User acceptance testing

### Production Rollout
- Feature flags for security features
- Gradual rollout with monitoring
- Rollback procedures documented
- Incident response team on standby

## Success Metrics

### Technical Metrics
- Zero critical security vulnerabilities in production
- Security test coverage >95%
- Performance impact <10%
- Security incident response time <1 hour

### Business Metrics
- Customer security satisfaction >90%
- Compliance audit pass rate 100%
- Security incident frequency <1 per quarter
- Developer productivity maintained

---

## **URGENT Implementation Priority Matrix**

### **CRITICAL (Week 1 - Block Production)**
| Vulnerability | File | Lines | Risk | Developer |
|---------------|------|-------|------|-----------|
| Unrestricted file access | `read.ts` | 60,86-87 | CRITICAL | Sr Dev 1 |
| Unrestricted file write | `write.ts` | 32,58,66 | CRITICAL | Sr Dev 1 |
| Unrestricted file edit | `edit.ts` | 132,161,176,218 | CRITICAL | Sr Dev 1 |
| Direct command execution | `bash.ts` | 82-84 | CRITICAL | Sr Dev 2 |

### **HIGH (Week 2)**
| Vulnerability | File | Lines | Risk | Developer |
|---------------|------|-------|------|-----------|
| Client-side API keys | `provider-keys-store.ts` | 18-19 | HIGH | Jr Dev 1 |
| Browser key exposure | `anthropic.ts` | 115,285-322 | HIGH | Jr Dev 1 |
| SSH no host verification | `ssh.ts` | 12-32,68-98 | HIGH | Jr Dev 2 |

## Task Assignment Matrix

| Task | Developer | Week | Dependencies | Priority |
|------|-----------|------|--------------|----------|
| 1.1 | Sr Dev 1 | 1 | None | CRITICAL |
| 1.2 | Sr Dev 2 | 1 | None | CRITICAL |
| 1.3 | Jr Dev 3 | 1 | None | HIGH |
| 2.1 | Jr Dev 1 | 2 | 1.1, 1.2, 1.3 | HIGH |
| 2.2 | Jr Dev 2 | 2 | 1.2, 1.3 | HIGH |
| 3.1 | Jr Dev 3 | 3 | 1.3, 2.1 | MEDIUM |
| 3.2 | Sr Dev 1 | 3 | All previous | MEDIUM |

## Risk Mitigation

### Technical Risks
- **Performance Impact**: Continuous benchmarking and optimization
- **Compatibility Issues**: Comprehensive testing and backward compatibility
- **Complexity**: Modular design and clear documentation

### Project Risks
- **Timeline Delays**: Buffer time in estimates and parallel task execution
- **Resource Constraints**: Cross-training and task reassignment flexibility
- **Quality Issues**: Code reviews and automated testing requirements

---

**Document History**:
- v1.0 (2025-11-13): Initial task breakdown - Security Team
- v1.1 (2025-11-14): Security assessment integration, specific implementation details with file paths and line numbers, environment variable API key strategy, urgent priority matrix - Security Team

**Contact Information**:
- Tech Lead: _________________________
- Security Lead: _________________________
- Project Manager: ______________________