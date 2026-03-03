# OpenClaw Research Findings: Security & File System (Q57-Q64)

**Date:** 2026-03-03  
**Source:** OpenClaw Documentation Database  
**Status:** Q57-Q64 Answered

---

## Security & Privacy (Q57-Q60)

### Q57: Secret Management

**Answer:**

**Storage:**
- Per-agent: `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`
- Environment variables: `OPENAI_API_KEY`, etc.
- Config `env` block: `skills.entries.<skillKey>.env`

**Rotation:**
- Automatic OAuth token refresh
- Uses refresh token to mint new access token
- File lock prevents collisions during refresh

**Plugin Access:**
- Secrets accessible if injected as environment variables
- Configured via `skills.entries.<skillKey>.env`

### Implications for Cognitive Controller

✅ **Secret Storage:**
- Can store Cognitive Controller secrets in auth-profiles.json
- Per-agent isolation for security

⚠️ **No Built-in Encryption:**
- Secrets stored in plaintext JSON
- Must rely on filesystem permissions

⚠️ **Plugin Access:**
- Must explicitly configure env vars for secret access
- No automatic secret injection


### Action Items

1. **Implement secure configuration:**

```typescript
interface CognitiveControllerSecrets {
  databaseEncryptionKey?: string;
  apiKeys?: Record<string, string>;
  webhookSecret?: string;
}

class SecretManager {
  private secretsPath: string;
  
  constructor(agentId: string) {
    this.secretsPath = path.join(
      os.homedir(),
      '.openclaw',
      'agents',
      agentId,
      'agent',
      'auth-profiles.json'
    );
  }
  
  async loadSecrets(): Promise<CognitiveControllerSecrets> {
    try {
      const content = await fs.readFile(this.secretsPath, 'utf-8');
      const authProfiles = JSON.parse(content);
      
      // Extract Cognitive Controller secrets
      return authProfiles['cognitive-controller'] || {};
    } catch (error) {
      if (error.code === 'ENOENT') {
        logger.info('No secrets file found, using defaults');
        return {};
      }
      throw error;
    }
  }
  
  async saveSecrets(secrets: CognitiveControllerSecrets): Promise<void> {
    // Read existing auth profiles
    let authProfiles: any = {};
    
    try {
      const content = await fs.readFile(this.secretsPath, 'utf-8');
      authProfiles = JSON.parse(content);
    } catch (error) {
      if (error.code !== 'ENOENT') throw error;
    }
    
    // Update Cognitive Controller secrets
    authProfiles['cognitive-controller'] = secrets;
    
    // Write back with restricted permissions
    await fs.writeFile(
      this.secretsPath,
      JSON.stringify(authProfiles, null, 2),
      { mode: 0o600 } // Owner read/write only
    );
    
    logger.info('Secrets saved securely');
  }
  
  async rotateSecret(key: string, newValue: string): Promise<void> {
    const secrets = await this.loadSecrets();
    
    if (!secrets.apiKeys) {
      secrets.apiKeys = {};
    }
    
    secrets.apiKeys[key] = newValue;
    
    await this.saveSecrets(secrets);
    
    logger.info(`Secret rotated: ${key}`);
  }
}
```

2. **Use environment variables for sensitive config:**

```typescript
class EnvironmentConfig {
  static getDatabaseEncryptionKey(): string | undefined {
    return process.env.CC_DATABASE_ENCRYPTION_KEY;
  }
  
  static getWebhookSecret(): string | undefined {
    return process.env.CC_WEBHOOK_SECRET;
  }
  
  static getApiKey(provider: string): string | undefined {
    const envVar = `CC_${provider.toUpperCase()}_API_KEY`;
    return process.env[envVar];
  }
  
  static loadFromEnv(): CognitiveControllerSecrets {
    return {
      databaseEncryptionKey: this.getDatabaseEncryptionKey(),
      webhookSecret: this.getWebhookSecret(),
      apiKeys: {
        openai: this.getApiKey('openai') || '',
        anthropic: this.getApiKey('anthropic') || ''
      }
    };
  }
}
```

3. **Configure skill environment variables:**

```json5
// In openclaw.json
{
  "skills": {
    "entries": {
      "cognitive-controller": {
        "enabled": true,
        "env": {
          "CC_DATABASE_ENCRYPTION_KEY": "${DATABASE_ENCRYPTION_KEY}",
          "CC_WEBHOOK_SECRET": "${WEBHOOK_SECRET}"
        }
      }
    }
  }
}
```

---

### Q58: Authentication Model

**Answer:**

**Gateway:**
- Control plane secured via `gateway.auth.token`
- WebSocket requires mandatory handshake
- First frame must be `connect` request with token

**Device Pairing:**
- New device IDs require pairing approval
- Gateway issues unique device token after approval

**Agent Auth:**
- Isolated per-agent
- Each agent maintains own model registry and credentials

**Webhooks:**
- Secured by `hooks.token`
- Provided as Bearer token or `x-openclaw-token` header

### Implications for Cognitive Controller

✅ **Gateway Integration:**
- Must authenticate with gateway.auth.token
- WebSocket handshake required

✅ **Per-Agent Auth:**
- Each agent has isolated credentials
- No cross-agent auth leakage

⚠️ **Webhook Security:**
- Must validate hooks.token for webhook endpoints
- Prevent unauthorized access

### Action Items

1. **Implement Gateway authentication:**

```typescript
import WebSocket from 'ws';

class GatewayClient {
  private ws: WebSocket | null = null;
  private authToken: string;
  
  constructor(authToken: string) {
    this.authToken = authToken;
  }
  
  async connect(): Promise<void> {
    return new Promise((resolve, reject) => {
      this.ws = new WebSocket('ws://127.0.0.1:18789');
      
      this.ws.on('open', () => {
        // Send mandatory handshake
        this.sendHandshake();
      });
      
      this.ws.on('message', (data) => {
        const message = JSON.parse(data.toString());
        
        if (message.type === 'connect_ack') {
          logger.info('Gateway connection established');
          resolve();
        } else if (message.type === 'error') {
          logger.error(`Gateway error: ${message.message}`);
          reject(new Error(message.message));
        }
      });
      
      this.ws.on('error', (error) => {
        logger.error(`WebSocket error: ${error.message}`);
        reject(error);
      });
    });
  }
  
  private sendHandshake(): void {
    if (!this.ws) return;
    
    const handshake = {
      type: 'connect',
      token: this.authToken,
      clientId: 'cognitive-controller',
      version: '1.0.0'
    };
    
    this.ws.send(JSON.stringify(handshake));
  }
  
  subscribe(eventType: string, handler: (event: any) => void): void {
    if (!this.ws) {
      throw new Error('Not connected to Gateway');
    }
    
    this.ws.on('message', (data) => {
      const message = JSON.parse(data.toString());
      
      if (message.type === eventType) {
        handler(message);
      }
    });
  }
  
  disconnect(): void {
    if (this.ws) {
      this.ws.close();
      this.ws = null;
    }
  }
}
```

2. **Implement webhook authentication:**

```typescript
import express from 'express';
import crypto from 'crypto';

class WebhookServer {
  private app: express.Application;
  private hooksToken: string;
  
  constructor(hooksToken: string) {
    this.app = express();
    this.hooksToken = hooksToken;
    
    this.setupMiddleware();
    this.setupRoutes();
  }
  
  private setupMiddleware(): void {
    this.app.use(express.json());
    
    // Authentication middleware
    this.app.use((req, res, next) => {
      const authHeader = req.headers.authorization;
      const tokenHeader = req.headers['x-openclaw-token'];
      
      let providedToken: string | null = null;
      
      if (authHeader && authHeader.startsWith('Bearer ')) {
        providedToken = authHeader.substring(7);
      } else if (tokenHeader) {
        providedToken = tokenHeader as string;
      }
      
      if (!providedToken) {
        logger.warn('Webhook request missing authentication');
        return res.status(401).json({ error: 'Authentication required' });
      }
      
      // Constant-time comparison to prevent timing attacks
      if (!this.constantTimeCompare(providedToken, this.hooksToken)) {
        logger.warn('Webhook request with invalid token');
        return res.status(403).json({ error: 'Invalid token' });
      }
      
      next();
    });
  }
  
  private constantTimeCompare(a: string, b: string): boolean {
    if (a.length !== b.length) return false;
    
    return crypto.timingSafeEqual(
      Buffer.from(a),
      Buffer.from(b)
    );
  }
  
  private setupRoutes(): void {
    // Webhook endpoint for external events
    this.app.post('/webhook/event', async (req, res) => {
      try {
        const event = req.body;
        
        logger.info(`Webhook event received: ${event.type}`);
        
        // Process event
        await this.processWebhookEvent(event);
        
        res.json({ status: 'ok' });
      } catch (error) {
        logger.error(`Webhook processing error: ${error.message}`);
        res.status(500).json({ error: 'Internal server error' });
      }
    });
    
    // Health check endpoint (no auth required)
    this.app.get('/health', (req, res) => {
      res.json({ status: 'healthy' });
    });
  }
  
  private async processWebhookEvent(event: any): Promise<void> {
    // Process webhook event
    // This could trigger snapshot creation, fact extraction, etc.
  }
  
  start(port: number): void {
    this.app.listen(port, () => {
      logger.info(`Webhook server listening on port ${port}`);
    });
  }
}
```

3. **Implement device pairing (if needed):**

```typescript
class DevicePairing {
  async requestPairing(deviceId: string): Promise<string> {
    logger.info(`Requesting pairing for device: ${deviceId}`);
    
    // Send pairing request to Gateway
    const response = await fetch('http://127.0.0.1:18789/api/pairing/request', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ deviceId })
    });
    
    if (!response.ok) {
      throw new Error(`Pairing request failed: ${response.statusText}`);
    }
    
    const data = await response.json();
    
    logger.info(`Pairing requested, awaiting approval. Code: ${data.pairingCode}`);
    
    return data.pairingCode;
  }
  
  async pollForApproval(pairingCode: string): Promise<string> {
    const maxAttempts = 60; // 5 minutes
    
    for (let i = 0; i < maxAttempts; i++) {
      const response = await fetch(`http://127.0.0.1:18789/api/pairing/status/${pairingCode}`);
      
      if (response.ok) {
        const data = await response.json();
        
        if (data.status === 'approved') {
          logger.info('Pairing approved!');
          return data.deviceToken;
        }
      }
      
      await this.sleep(5000); // Wait 5 seconds
    }
    
    throw new Error('Pairing timeout: no approval received');
  }
  
  private sleep(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}
```

---

### Q59: Data Privacy Guarantees

**Answer:**

**Encryption at Rest:**
- **No automatic filesystem encryption**
- Documentation recommends: encrypt backups, treat state directory as production secret

**Encryption in Transit:**
- Remote access via Tailscale VPN or SSH tunnels
- TLS and optional pinning for WebSocket in remote setups

**Retention:**
- Session Pruning mechanism
- Cron jobs: `sessionRetention` flag (e.g., "24h")

### Implications for Cognitive Controller

⚠️ **No Built-in Encryption:**
- Must implement our own encryption if needed
- Filesystem permissions are primary security

✅ **Transit Security:**
- Can use TLS for remote access
- VPN recommended for production

✅ **Retention Control:**
- Can implement retention policies
- Align with OpenClaw's session pruning

### Action Items

1. **Implement optional database encryption:**

```typescript
import crypto from 'crypto';

class EncryptedStorage {
  private encryptionKey: Buffer | null = null;
  private algorithm = 'aes-256-gcm';
  
  constructor(encryptionKey?: string) {
    if (encryptionKey) {
      // Derive 32-byte key from provided key
      this.encryptionKey = crypto.scryptSync(encryptionKey, 'salt', 32);
    }
  }
  
  encrypt(data: string): string {
    if (!this.encryptionKey) {
      return data; // No encryption
    }
    
    const iv = crypto.randomBytes(16);
    const cipher = crypto.createCipheriv(this.algorithm, this.encryptionKey, iv);
    
    let encrypted = cipher.update(data, 'utf8', 'hex');
    encrypted += cipher.final('hex');
    
    const authTag = cipher.getAuthTag();
    
    // Return: iv + authTag + encrypted
    return iv.toString('hex') + authTag.toString('hex') + encrypted;
  }
  
  decrypt(encryptedData: string): string {
    if (!this.encryptionKey) {
      return encryptedData; // No encryption
    }
    
    // Extract: iv (32 chars) + authTag (32 chars) + encrypted
    const iv = Buffer.from(encryptedData.slice(0, 32), 'hex');
    const authTag = Buffer.from(encryptedData.slice(32, 64), 'hex');
    const encrypted = encryptedData.slice(64);
    
    const decipher = crypto.createDecipheriv(this.algorithm, this.encryptionKey, iv);
    decipher.setAuthTag(authTag);
    
    let decrypted = decipher.update(encrypted, 'hex', 'utf8');
    decrypted += decipher.final('utf8');
    
    return decrypted;
  }
  
  async saveEncrypted(filepath: string, data: any): Promise<void> {
    const json = JSON.stringify(data);
    const encrypted = this.encrypt(json);
    
    await fs.writeFile(filepath, encrypted, { mode: 0o600 });
  }
  
  async loadEncrypted(filepath: string): Promise<any> {
    const encrypted = await fs.readFile(filepath, 'utf-8');
    const decrypted = this.decrypt(encrypted);
    
    return JSON.parse(decrypted);
  }
}

// Usage
const encryptionKey = EnvironmentConfig.getDatabaseEncryptionKey();
const encryptedStorage = new EncryptedStorage(encryptionKey);
```

2. **Implement retention policies:**

```typescript
class RetentionPolicy {
  private policies: Map<string, RetentionConfig> = new Map();
  
  setPolicy(dataType: string, config: RetentionConfig): void {
    this.policies.set(dataType, config);
    logger.info(`Retention policy set for ${dataType}: ${config.retentionDays} days`);
  }
  
  async enforceRetention(agentId: string): Promise<void> {
    logger.info(`Enforcing retention policies for agent: ${agentId}`);
    
    // Enforce snapshot retention
    const snapshotPolicy = this.policies.get('snapshots');
    if (snapshotPolicy) {
      await this.pruneSnapshots(agentId, snapshotPolicy);
    }
    
    // Enforce fact retention
    const factPolicy = this.policies.get('facts');
    if (factPolicy) {
      await this.pruneFacts(agentId, factPolicy);
    }
    
    // Enforce roll-up retention
    const rollUpPolicy = this.policies.get('rollups');
    if (rollUpPolicy) {
      await this.pruneRollUps(agentId, rollUpPolicy);
    }
  }
  
  private async pruneSnapshots(agentId: string, policy: RetentionConfig): Promise<void> {
    const cutoffDate = new Date();
    cutoffDate.setDate(cutoffDate.getDate() - policy.retentionDays);
    
    const deleted = await sqliteStore.deleteSnapshotsBefore(agentId, cutoffDate);
    
    logger.info(`Pruned ${deleted} snapshots older than ${policy.retentionDays} days`);
  }
  
  private async pruneFacts(agentId: string, policy: RetentionConfig): Promise<void> {
    const cutoffDate = new Date();
    cutoffDate.setDate(cutoffDate.getDate() - policy.retentionDays);
    
    const deleted = await sqliteStore.deleteFactsBefore(agentId, cutoffDate);
    
    logger.info(`Pruned ${deleted} facts older than ${policy.retentionDays} days`);
  }
  
  private async pruneRollUps(agentId: string, policy: RetentionConfig): Promise<void> {
    const rollUpDir = pathManager.getRollUpDir(agentId);
    const files = await fs.readdir(rollUpDir);
    
    const cutoffDate = new Date();
    cutoffDate.setDate(cutoffDate.getDate() - policy.retentionDays);
    
    let deleted = 0;
    
    for (const file of files) {
      const filepath = path.join(rollUpDir, file);
      const stats = await fs.stat(filepath);
      
      if (stats.mtime < cutoffDate) {
        await fs.unlink(filepath);
        deleted++;
      }
    }
    
    logger.info(`Pruned ${deleted} roll-up files older than ${policy.retentionDays} days`);
  }
}

interface RetentionConfig {
  retentionDays: number;
  archiveBeforeDelete?: boolean;
}

const retentionPolicy = new RetentionPolicy();

// Set default policies
retentionPolicy.setPolicy('snapshots', { retentionDays: 90 });
retentionPolicy.setPolicy('facts', { retentionDays: 180 });
retentionPolicy.setPolicy('rollups', { retentionDays: 365 });
```

3. **Implement secure backup:**

```typescript
class SecureBackup {
  async createBackup(agentId: string): Promise<string> {
    logger.info(`Creating secure backup for agent: ${agentId}`);
    
    const timestamp = new Date().toISOString().replace(/[:.]/g, '-');
    const backupDir = path.join(os.homedir(), '.openclaw', 'backups', agentId);
    const backupPath = path.join(backupDir, `backup-${timestamp}.tar.gz`);
    
    await fs.mkdir(backupDir, { recursive: true });
    
    // Backup database
    const dbPath = pathManager.getDatabasePath(agentId);
    const rollUpDir = pathManager.getRollUpDir(agentId);
    
    // Create tar.gz archive
    await this.createArchive(backupPath, [dbPath, rollUpDir]);
    
    // Encrypt backup if encryption key available
    const encryptionKey = EnvironmentConfig.getDatabaseEncryptionKey();
    if (encryptionKey) {
      await this.encryptBackup(backupPath, encryptionKey);
    }
    
    logger.info(`Backup created: ${backupPath}`);
    
    return backupPath;
  }
  
  private async createArchive(outputPath: string, paths: string[]): Promise<void> {
    // Use tar command to create archive
    const pathsStr = paths.join(' ');
    await executePwsh({ command: `tar -czf ${outputPath} ${pathsStr}` });
  }
  
  private async encryptBackup(backupPath: string, encryptionKey: string): Promise<void> {
    const encryptedPath = `${backupPath}.enc`;
    
    // Read backup
    const data = await fs.readFile(backupPath);
    
    // Encrypt
    const storage = new EncryptedStorage(encryptionKey);
    const encrypted = storage.encrypt(data.toString('base64'));
    
    // Write encrypted backup
    await fs.writeFile(encryptedPath, encrypted, { mode: 0o600 });
    
    // Delete unencrypted backup
    await fs.unlink(backupPath);
    
    logger.info(`Backup encrypted: ${encryptedPath}`);
  }
}

const secureBackup = new SecureBackup();
```

---

### Q60: PII Handling

**Answer:**

**PII/Redaction:**
- **No dedicated PII redaction engine**
- Hook payloads wrapped with `external-content` safety boundaries
- Isolates untrusted data by default

**Compliance:**
- **No specific GDPR or CCPA compliance features mentioned**

**Sensitivity:**
- Documentation advises against storing secrets or sensitive payloads in:
  - Webhook logs
  - HEARTBEAT.md

### Implications for Cognitive Controller

⚠️ **No Built-in PII Redaction:**
- Must implement our own if needed
- Critical for compliance

⚠️ **No Compliance Features:**
- Must implement GDPR/CCPA features ourselves
- Right to deletion, data export, etc.

✅ **Safety Boundaries:**
- Can use external-content pattern for untrusted data
- Prevents injection attacks

### Action Items

1. **Implement PII detection and redaction:**

```typescript
class PIIRedactor {
  private patterns: Map<string, RegExp> = new Map([
    ['email', /\b[A-Za-z0-9._%+-]+@[A-z0-9.-]+\.[A-Z|a-z]{2,}\b/g],
    ['phone', /\b(\+\d{1,3}[-.]?)?\(?\d{3}\)?[-.]?\d{3}[-.]?\d{4}\b/g],
    ['ssn', /\b\d{3}-\d{2}-\d{4}\b/g],
    ['credit_card', /\b\d{4}[-\s]?\d{4}[-\s]?\d{4}[-\s]?\d{4}\b/g],
    ['ip_address', /\b\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}\b/g]
  ]);
  
  detectPII(text: string): PIIDetection[] {
    const detections: PIIDetection[] = [];
    
    for (const [type, pattern] of this.patterns.entries()) {
      const matches = text.matchAll(pattern);
      
      for (const match of matches) {
        detections.push({
          type,
          value: match[0],
          start: match.index!,
          end: match.index! + match[0].length
        });
      }
    }
    
    return detections;
  }
  
  redact(text: string, replacement: string = '[REDACTED]'): string {
    let redacted = text;
    
    for (const pattern of this.patterns.values()) {
      redacted = redacted.replace(pattern, replacement);
    }
    
    return redacted;
  }
  
  redactWithType(text: string): string {
    let redacted = text;
    
    for (const [type, pattern] of this.patterns.entries()) {
      redacted = redacted.replace(pattern, `[${type.toUpperCase()}_REDACTED]`);
    }
    
    return redacted;
  }
}

interface PIIDetection {
  type: string;
  value: string;
  start: number;
  end: number;
}

const piiRedactor = new PIIRedactor();
```

2. **Implement GDPR compliance features:**

```typescript
class GDPRCompliance {
  async exportUserData(agentId: string): Promise<string> {
    logger.info(`Exporting user data for agent: ${agentId} (GDPR request)`);
    
    // Collect all data
    const snapshots = await sqliteStore.getAllSnapshots(agentId);
    const facts = await sqliteStore.getAllFacts(agentId);
    const rollUps = await this.getRollUpFiles(agentId);
    
    const exportData = {
      agentId,
      exportDate: new Date().toISOString(),
      snapshots,
      facts,
      rollUps
    };
    
    // Write to file
    const exportPath = path.join(
      pathManager.getCognitiveControllerDir(agentId),
      `gdpr-export-${Date.now()}.json`
    );
    
    await fs.writeFile(exportPath, JSON.stringify(exportData, null, 2));
    
    logger.info(`User data exported: ${exportPath}`);
    
    return exportPath;
  }
  
  async deleteUserData(agentId: string): Promise<void> {
    logger.info(`Deleting user data for agent: ${agentId} (GDPR request)`);
    
    // Delete all snapshots
    await sqliteStore.deleteAllSnapshots(agentId);
    
    // Delete all facts
    await sqliteStore.deleteAllFacts(agentId);
    
    // Delete roll-up files
    const rollUpDir = pathManager.getRollUpDir(agentId);
    await fs.rm(rollUpDir, { recursive: true, force: true });
    
    // Recreate empty directory
    await fs.mkdir(rollUpDir, { recursive: true });
    
    logger.info(`User data deleted for agent: ${agentId}`);
  }
  
  async anonymizeUserData(agentId: string): Promise<void> {
    logger.info(`Anonymizing user data for agent: ${agentId}`);
    
    // Anonymize facts
    const facts = await sqliteStore.getAllFacts(agentId);
    
    for (const fact of facts) {
      const anonymized = piiRedactor.redact(fact.content);
      await sqliteStore.updateFact(fact.fact_id, { content: anonymized });
    }
    
    // Anonymize snapshots
    const snapshots = await sqliteStore.getAllSnapshots(agentId);
    
    for (const snapshot of snapshots) {
      const anonymized = this.anonymizeSnapshot(snapshot);
      await sqliteStore.updateSnapshot(snapshot.handshake.snapshot_id, anonymized);
    }
    
    logger.info(`User data anonymized for agent: ${agentId}`);
  }
  
  private anonymizeSnapshot(snapshot: Snapshot): Snapshot {
    return {
      ...snapshot,
      objective_now: piiRedactor.redact(snapshot.objective_now),
      recent_facts: snapshot.recent_facts.map(f => ({
        ...f,
        content: piiRedactor.redact(f.content)
      })),
      active_threads: snapshot.active_threads.map(t => ({
        ...t,
        summary: piiRedactor.redact(t.summary)
      }))
    };
  }
  
  private async getRollUpFiles(agentId: string): Promise<string[]> {
    const rollUpDir = pathManager.getRollUpDir(agentId);
    const files = await fs.readdir(rollUpDir);
    
    const contents: string[] = [];
    
    for (const file of files) {
      const content = await fs.readFile(path.join(rollUpDir, file), 'utf-8');
      contents.push(content);
    }
    
    return contents;
  }
}

const gdprCompliance = new GDPRCompliance();
```

3. **Implement external-content safety boundaries:**

```typescript
class SafetyBoundaries {
  wrapUntrustedContent(content: string, source: string): string {
    return `<!-- external-content: ${source} -->
${content}
<!-- /external-content -->`;
  }
  
  extractUntrustedContent(wrapped: string): { content: string; source: string } | null {
    const match = wrapped.match(/<!-- external-content: (.+?) -->\n([\s\S]+?)\n<!-- \/external-content -->/);
    
    if (!match) return null;
    
    return {
      source: match[1],
      content: match[2]
    };
  }
  
  sanitizeUntrustedContent(content: string): string {
    // Remove potentially dangerous content
    let sanitized = content;
    
    // Remove script tags
    sanitized = sanitized.replace(/<script\b[^<]*(?:(?!<\/script>)<[^<]*)*<\/script>/gi, '');
    
    // Remove event handlers
    sanitized = sanitized.replace(/on\w+\s*=\s*["'][^"']*["']/gi, '');
    
    // Redact PII
    sanitized = piiRedactor.redact(sanitized);
    
    return sanitized;
  }
}

const safetyBoundaries = new SafetyBoundaries();
```

---

## File System & Storage (Q61-Q64)

### Q61: File System Constraints

**Answer:**

**Required Structure:**
- `AGENTS.md` - Instructions
- `SOUL.md` - Persona
- `USER.md` - Profile
- `IDENTITY.md` - Agent name/vibe
- `TOOLS.md` - Conventions

**Naming Conventions:**
- Daily logs: `YYYY-MM-DD.md` pattern (for indexing)

**Reserved Filenames:**
- `BOOTSTRAP.md` - One-time first-run ritual (deleted after completion)
- `HEARTBEAT.md` - Periodic checklist tasks

### Implications for Cognitive Controller

✅ **Clear Structure:**
- Well-defined workspace layout
- Can integrate with existing files

⚠️ **Reserved Names:**
- Must avoid `BOOTSTRAP.md` and `HEARTBEAT.md`
- Use different names for our files

✅ **Naming Patterns:**
- Can follow YYYY-MM-DD pattern for time-based files
- Aligns with OpenClaw conventions

### Action Items

1. **Define Cognitive Controller file structure:**

```typescript
class FileStructure {
  static readonly REQUIRED_FILES = {
    // OpenClaw required files
    AGENTS: 'AGENTS.md',
    SOUL: 'SOUL.md',
    USER: 'USER.md',
    IDENTITY: 'IDENTITY.md',
    TOOLS: 'TOOLS.md',
    
    // Cognitive Controller files (avoid reserved names)
    CC_MEMORY: 'COGNITIVE_CONTROLLER.md',
    CC_CONSTRAINTS: 'CONSTRAINTS.md',
    CC_STATUS: 'CC_STATUS.md'
  };
  
  static readonly RESERVED_FILES = [
    'BOOTSTRAP.md',
    'HEARTBEAT.md'
  ];
  
  static readonly DIRECTORIES = {
    MEMORY: 'memory',
    ROLLUPS: 'memory/rollups',
    CC_DATA: '.cognitive-controller',
    DAILY_LOGS: 'memory/daily'
  };
  
  static isReservedFilename(filename: string): boolean {
    return this.RESERVED_FILES.includes(filename);
  }
  
  static getDailyLogFilename(date: Date): string {
    const year = date.getFullYear();
    const month = String(date.getMonth() + 1).padStart(2, '0');
    const day = String(date.getDate()).padStart(2, '0');
    
    return `${year}-${month}-${day}.md`;
  }
  
  static async initializeWorkspace(agentId: string): Promise<void> {
    const workspace = pathManager.getWorkspace(agentId);
    
    // Create directories
    for (const dir of Object.values(this.DIRECTORIES)) {
      await fs.mkdir(path.join(workspace, dir), { recursive: true });
    }
    
    // Create Cognitive Controller files if they don't exist
    await this.createFileIfNotExists(
      path.join(workspace, this.REQUIRED_FILES.CC_MEMORY),
      this.getDefaultCCMemoryContent()
    );
    
    await this.createFileIfNotExists(
      path.join(workspace, this.REQUIRED_FILES.CC_CONSTRAINTS),
      this.getDefaultConstraintsContent()
    );
    
    await this.createFileIfNotExists(
      path.join(workspace, this.REQUIRED_FILES.CC_STATUS),
      this.getDefaultStatusContent()
    );
    
    logger.info(`Workspace initialized for agent: ${agentId}`);
  }
  
  private static async createFileIfNotExists(filepath: string, content: string): Promise<void> {
    try {
      await fs.access(filepath);
      // File exists, don't overwrite
    } catch (error) {
      if (error.code === 'ENOENT') {
        await fs.writeFile(filepath, content, 'utf-8');
        logger.info(`Created file: ${filepath}`);
      } else {
        throw error;
      }
    }
  }
  
  private static getDefaultCCMemoryContent(): string {
    return `# Cognitive Controller Memory

## Status
Initialized and ready.

## Active Snapshots
None yet.

## Recent Activity
- Cognitive Controller initialized
`;
  }
  
  private static getDefaultConstraintsContent(): string {
    return `# Hard Constraints

> These constraints must NEVER be paraphrased or modified.

- Preserve all constraints verbatim
- Never paraphrase hard rules
- Maintain bounded memory (1-10 snapshots per thread)
- Snapshot token budget: 500 tokens
- Create snapshot on memory flush
`;
  }
  
  private static getDefaultStatusContent(): string {
    return `# Cognitive Controller Status

**Last Updated:** ${new Date().toISOString()}

## Health
- Database: OK
- Snapshots: 0
- Facts: 0

## Configuration
- Snapshot Budget: 500 tokens
- Max Active Threads: 7
- Retention: 90 days
`;
  }
}
```

2. **Implement daily log management:**

```typescript
class DailyLogManager {
  async createDailyLog(agentId: string, date: Date = new Date()): Promise<string> {
    const workspace = pathManager.getWorkspace(agentId);
    const dailyLogDir = path.join(workspace, FileStructure.DIRECTORIES.DAILY_LOGS);
    
    await fs.mkdir(dailyLogDir, { recursive: true });
    
    const filename = FileStructure.getDailyLogFilename(date);
    const filepath = path.join(dailyLogDir, filename);
    
    // Check if log already exists
    try {
      await fs.access(filepath);
      logger.info(`Daily log already exists: ${filename}`);
      return filepath;
    } catch (error) {
      if (error.code !== 'ENOENT') throw error;
    }
    
    // Create new daily log
    const content = this.generateDailyLogContent(date);
    await fs.writeFile(filepath, content, 'utf-8');
    
    logger.info(`Created daily log: ${filename}`);
    
    return filepath;
  }
  
  private generateDailyLogContent(date: Date): string {
    const dateStr = date.toISOString().split('T')[0];
    
    return `# Daily Log: ${dateStr}

## Summary
No activity yet.

## Snapshots Created
None.

## Facts Captured
None.

## Decisions Made
None.

## Open Questions
None.
`;
  }
  
  async appendToDailyLog(agentId: string, section: string, content: string): Promise<void> {
    const filepath = await this.createDailyLog(agentId);
    
    // Read current content
    const currentContent = await fs.readFile(filepath, 'utf-8');
    
    // Find section
    const sectionHeader = `## ${section}`;
    const sectionIndex = currentContent.indexOf(sectionHeader);
    
    if (sectionIndex === -1) {
      logger.warn(`Section not found in daily log: ${section}`);
      return;
    }
    
    // Find next section or end of file
    const nextSectionIndex = currentContent.indexOf('\n## ', sectionIndex + sectionHeader.length);
    const insertIndex = nextSectionIndex === -1 ? currentContent.length : nextSectionIndex;
    
    // Insert content
    const before = currentContent.slice(0, insertIndex);
    const after = currentContent.slice(insertIndex);
    const newContent = `${before}\n${content}\n${after}`;
    
    await fs.writeFile(filepath, newContent, 'utf-8');
    
    logger.info(`Appended to daily log section: ${section}`);
  }
}

const dailyLogManager = new DailyLogManager();
```

---

### Q62: File Size Limits

**Answer:**

**Memory Files:**
- Per-file limit: 20,000 characters (`bootstrapMaxChars`)
- Total limit: 150,000 characters (`bootstrapTotalMaxChars`)
- Keeps prompts lean

**Total Workspace:**
- **No explicit total size cap mentioned**

**Retrieval:**
- Clamped by `maxResults` and `maxSnippetChars`

**Quota Enforcement:**
- **No quota system for disk usage mentioned**

### Implications for Cognitive Controller

✅ **Bootstrap Limits:**
- 20K per file is generous for snapshots
- 150K total allows multiple files

⚠️ **No Workspace Limit:**
- Could grow unbounded
- Must implement our own limits

✅ **Retrieval Limits:**
- Can control snippet sizes
- Prevents overwhelming context

### Action Items

1. **Implement file size validation:**

```typescript
class FileSizeValidator {
  private readonly MAX_FILE_SIZE = 20000; // 20K characters
  private readonly MAX_TOTAL_SIZE = 150000; // 150K characters
  
  async validateFile(filepath: string): Promise<ValidationResult> {
    const content = await fs.readFile(filepath, 'utf-8');
    const size = content.length;
    
    if (size > this.MAX_FILE_SIZE) {
      return {
        valid: false,
        size,
        limit: this.MAX_FILE_SIZE,
        message: `File exceeds size limit: ${size} > ${this.MAX_FILE_SIZE}`
      };
    }
    
    return {
      valid: true,
      size,
      limit: this.MAX_FILE_SIZE,
      message: 'File size OK'
    };
  }
  
  async validateWorkspace(agentId: string): Promise<WorkspaceValidation> {
    const workspace = pathManager.getWorkspace(agentId);
    
    // Get all Markdown files
    const files = await this.getAllMarkdownFiles(workspace);
    
    let totalSize = 0;
    const fileValidations: FileValidation[] = [];
    
    for (const file of files) {
      const content = await fs.readFile(file, 'utf-8');
      const size = content.length;
      
      totalSize += size;
      
      fileValidations.push({
        path: file,
        size,
        valid: size <= this.MAX_FILE_SIZE
      });
    }
    
    return {
      totalSize,
      totalLimit: this.MAX_TOTAL_SIZE,
      valid: totalSize <= this.MAX_TOTAL_SIZE,
      files: fileValidations
    };
  }
  
  private async getAllMarkdownFiles(dir: string): Promise<string[]> {
    const files: string[] = [];
    
    const entries = await fs.readdir(dir, { withFileTypes: true });
    
    for (const entry of entries) {
      const fullPath = path.join(dir, entry.name);
      
      if (entry.isDirectory()) {
        // Skip .cognitive-controller directory
        if (entry.name === '.cognitive-controller') continue;
        
        const subFiles = await this.getAllMarkdownFiles(fullPath);
        files.push(...subFiles);
      } else if (entry.name.endsWith('.md')) {
        files.push(fullPath);
      }
    }
    
    return files;
  }
  
  async trimFileToLimit(filepath: string): Promise<void> {
    const content = await fs.readFile(filepath, 'utf-8');
    
    if (content.length <= this.MAX_FILE_SIZE) {
      return; // No trimming needed
    }
    
    logger.warn(`Trimming file to size limit: ${filepath}`);
    
    // Trim from the middle, keep beginning and end
    const keepSize = Math.floor(this.MAX_FILE_SIZE / 2);
    const beginning = content.slice(0, keepSize);
    const end = content.slice(-keepSize);
    
    const trimmed = `${beginning}\n\n... [TRIMMED ${content.length - this.MAX_FILE_SIZE} characters] ...\n\n${end}`;
    
    await fs.writeFile(filepath, trimmed, 'utf-8');
  }
}

interface ValidationResult {
  valid: boolean;
  size: number;
  limit: number;
  message: string;
}

interface WorkspaceValidation {
  totalSize: number;
  totalLimit: number;
  valid: boolean;
  files: FileValidation[];
}

interface FileValidation {
  path: string;
  size: number;
  valid: boolean;
}

const fileSizeValidator = new FileSizeValidator();
```

2. **Implement workspace quota management:**

```typescript
class WorkspaceQuotaManager {
  private readonly DEFAULT_QUOTA = 100 * 1024 * 1024; // 100MB
  private quotas: Map<string, number> = new Map();
  
  setQuota(agentId: string, quotaBytes: number): void {
    this.quotas.set(agentId, quotaBytes);
    logger.info(`Quota set for agent ${agentId}: ${quotaBytes} bytes`);
  }
  
  getQuota(agentId: string): number {
    return this.quotas.get(agentId) || this.DEFAULT_QUOTA;
  }
  
  async checkQuota(agentId: string): Promise<QuotaStatus> {
    const quota = this.getQuota(agentId);
    const usage = await this.calculateUsage(agentId);
    
    return {
      quota,
      usage,
      available: quota - usage,
      percentUsed: (usage / quota) * 100,
      exceeded: usage > quota
    };
  }
  
  private async calculateUsage(agentId: string): Promise<number> {
    const ccDir = pathManager.getCognitiveControllerDir(agentId);
    const rollUpDir = pathManager.getRollUpDir(agentId);
    
    const ccSize = await this.getDirectorySize(ccDir);
    const rollUpSize = await this.getDirectorySize(rollUpDir);
    
    return ccSize + rollUpSize;
  }
  
  private async getDirectorySize(dir: string): Promise<number> {
    let totalSize = 0;
    
    try {
      const entries = await fs.readdir(dir, { withFileTypes: true });
      
      for (const entry of entries) {
        const fullPath = path.join(dir, entry.name);
        
        if (entry.isDirectory()) {
          totalSize += await this.getDirectorySize(fullPath);
        } else {
          const stats = await fs.stat(fullPath);
          totalSize += stats.size;
        }
      }
    } catch (error) {
      if (error.code !== 'ENOENT') throw error;
    }
    
    return totalSize;
  }
  
  async enforceQuota(agentId: string): Promise<void> {
    const status = await this.checkQuota(agentId);
    
    if (!status.exceeded) {
      return; // Within quota
    }
    
    logger.warn(`Quota exceeded for agent ${agentId}: ${status.usage} / ${status.quota} bytes`);
    
    // Trigger cleanup
    await this.cleanupOldData(agentId, status.usage - status.quota);
  }
  
  private async cleanupOldData(agentId: string, bytesToFree: number): Promise<void> {
    logger.info(`Cleaning up ${bytesToFree} bytes for agent: ${agentId}`);
    
    // Delete oldest snapshots first
    let freed = 0;
    const snapshots = await sqliteStore.getOldestSnapshots(agentId, 100);
    
    for (const snapshot of snapshots) {
      if (freed >= bytesToFree) break;
      
      const size = JSON.stringify(snapshot).length;
      await sqliteStore.deleteSnapshot(snapshot.handshake.snapshot_id);
      freed += size;
    }
    
    // Delete oldest roll-ups if needed
    if (freed < bytesToFree) {
      const rollUpDir = pathManager.getRollUpDir(agentId);
      const files = await fs.readdir(rollUpDir);
      
      // Sort by modification time
      const fileStats = await Promise.all(
        files.map(async (file) => {
          const filepath = path.join(rollUpDir, file);
          const stats = await fs.stat(filepath);
          return { file, filepath, mtime: stats.mtime, size: stats.size };
        })
      );
      
      fileStats.sort((a, b) => a.mtime.getTime() - b.mtime.getTime());
      
      for (const { filepath, size } of fileStats) {
        if (freed >= bytesToFree) break;
        
        await fs.unlink(filepath);
        freed += size;
      }
    }
    
    logger.info(`Cleaned up ${freed} bytes for agent: ${agentId}`);
  }
}

interface QuotaStatus {
  quota: number;
  usage: number;
  available: number;
  percentUsed: number;
  exceeded: boolean;
}

const quotaManager = new WorkspaceQuotaManager();
```

---

### Q63: File Operation Monitoring

**Answer:**

**Watcher:**
- Real-time file watcher for `MEMORY.md` and `memory/` directory

**Subscriptions:**
- Gateway can watch skill folders (`load.watch: true`)
- Refreshes skills snapshot when files change

**Debouncing:**
- Memory indexing: 1.5s debounce
- Skills: 250ms debounce
- Prevents excessive re-indexing during rapid writes

### Implications for Cognitive Controller

✅ **Real-time Monitoring:**
- Can detect file changes immediately
- Useful for triggering snapshot creation

✅ **Debouncing:**
- Prevents excessive processing
- Should implement similar pattern

### Action Items

1. **Implement file watcher:**

```typescript
import chokidar from 'chokidar';

class FileWatcher {
  private watchers: Map<string, chokidar.FSWatcher> = new Map();
  private debounceTimers: Map<string, NodeJS.Timeout> = new Map();
  private readonly DEBOUNCE_MS = 1500; // Match OpenClaw's 1.5s
  
  watchMemoryFiles(agentId: string, handler: (filepath: string) => void): void {
    const workspace = pathManager.getWorkspace(agentId);
    const memoryDir = path.join(workspace, 'memory');
    
    const watcher = chokidar.watch([
      path.join(workspace, 'MEMORY.md'),
      path.join(memoryDir, '**/*.md')
    ], {
      persistent: true,
      ignoreInitial: true,
      awaitWriteFinish: {
        stabilityThreshold: 500,
        pollInterval: 100
      }
    });
    
    watcher.on('change', (filepath) => {
      this.debouncedHandler(agentId, filepath, handler);
    });
    
    watcher.on('add', (filepath) => {
      this.debouncedHandler(agentId, filepath, handler);
    });
    
    this.watchers.set(agentId, watcher);
    
    logger.info(`File watcher started for agent: ${agentId}`);
  }
  
  private debouncedHandler(
    agentId: string,
    filepath: string,
    handler: (filepath: string) => void
  ): void {
    const key = `${agentId}:${filepath}`;
    
    // Clear existing timer
    const existingTimer = this.debounceTimers.get(key);
    if (existingTimer) {
      clearTimeout(existingTimer);
    }
    
    // Set new timer
    const timer = setTimeout(() => {
      logger.info(`File changed (debounced): ${filepath}`);
      handler(filepath);
      this.debounceTimers.delete(key);
    }, this.DEBOUNCE_MS);
    
    this.debounceTimers.set(key, timer);
  }
  
  stopWatching(agentId: string): void {
    const watcher = this.watchers.get(agentId);
    
    if (watcher) {
      watcher.close();
      this.watchers.delete(agentId);
      logger.info(`File watcher stopped for agent: ${agentId}`);
    }
  }
  
  stopAll(): void {
    for (const [agentId, watcher] of this.watchers.entries()) {
      watcher.close();
      logger.info(`File watcher stopped for agent: ${agentId}`);
    }
    
    this.watchers.clear();
    
    // Clear all debounce timers
    for (const timer of this.debounceTimers.values()) {
      clearTimeout(timer);
    }
    
    this.debounceTimers.clear();
  }
}

const fileWatcher = new FileWatcher();
```

2. **Integrate with snapshot creation:**

```typescript
class FileChangeHandler {
  async onMemoryFileChanged(agentId: string, filepath: string): Promise<void> {
    logger.info(`Memory file changed for agent ${agentId}: ${filepath}`);
    
    // Extract facts from changed file
    const content = await fs.readFile(filepath, 'utf-8');
    const facts = await factExtractor.extract(content);
    
    // Store facts
    for (const fact of facts) {
      await sqliteStore.insertFact({
        fact_id: generateUUID(),
        timestamp: new Date().toISOString(),
        source_type: 'file',
        source_id: filepath,
        content: fact.content,
        tags: ['file-change', path.basename(filepath)],
        confidence: 0.9
      });
    }
    
    logger.info(`Extracted ${facts.length} facts from file change`);
    
    // Check if snapshot should be created
    const shouldCreateSnapshot = await this.shouldCreateSnapshot(agentId);
    
    if (shouldCreateSnapshot) {
      logger.info('Triggering snapshot creation due to file changes');
      await snapshotAssembly.createSnapshot({ agentId, trigger: 'file_change' });
    }
  }
  
  private async shouldCreateSnapshot(agentId: string): Promise<boolean> {
    // Get last snapshot
    const lastSnapshot = await sqliteStore.getLatestSnapshot(agentId);
    
    if (!lastSnapshot) {
      return true; // No snapshot yet
    }
    
    // Check time since last snapshot
    const lastSnapshotTime = new Date(lastSnapshot.handshake.created_at).getTime();
    const timeSinceLastSnapshot = Date.now() - lastSnapshotTime;
    
    // Create snapshot if > 1 hour since last
    if (timeSinceLastSnapshot > 3600000) {
      return true;
    }
    
    // Check number of new facts
    const newFactCount = await sqliteStore.getFactCountSince(agentId, lastSnapshotTime);
    
    // Create snapshot if > 50 new facts
    return newFactCount > 50;
  }
}

const fileChangeHandler = new FileChangeHandler();

// Initialize file watching for all agents
async function initializeFileWatching(): Promise<void> {
  const config = await gateway.getConfig();
  const agents = config.agents.list
    .filter(a => a.skills?.['cognitive-controller']?.enabled)
    .map(a => a.id);
  
  for (const agentId of agents) {
    fileWatcher.watchMemoryFiles(agentId, (filepath) => {
      fileChangeHandler.onMemoryFileChanged(agentId, filepath);
    });
  }
}
```

---

### Q64: File Locking Strategy

**Answer:**

**Writing:**
- OAuth token refresh occurs under file lock
- Ensures data integrity

**Concurrent Access:**
- Gateway is only process allowed to own WhatsApp session per host
- Prevents conflicts

**Conflict Prevention:**
- Control-plane write RPCs rate-limited (3 per 60s)
- Coalescing mechanism for restarts

### Implications for Cognitive Controller

✅ **File Locking:**
- Should use file locks for critical writes
- Prevents corruption

⚠️ **Single Process:**
- Must ensure only one Cognitive Controller instance per agent
- Prevent concurrent writes

### Action Items

1. **Implement file locking:**

```typescript
import lockfile from 'proper-lockfile';

class FileLockManager {
  private locks: Map<string, () => Promise<void>> = new Map();
  
  async withLock<T>(
    filepath: string,
    operation: () => Promise<T>
  ): Promise<T> {
    // Acquire lock
    const release = await lockfile.lock(filepath, {
      retries: {
        retries: 10,
        minTimeout: 100,
        maxTimeout: 1000
      }
    });
    
    this.locks.set(filepath, release);
    
    try {
      // Perform operation
      return await operation();
    } finally {
      // Release lock
      await release();
      this.locks.delete(filepath);
    }
  }
  
  async releaseAll(): Promise<void> {
    for (const [filepath, release] of this.locks.entries()) {
      try {
        await release();
        logger.info(`Released lock: ${filepath}`);
      } catch (error) {
        logger.error(`Failed to release lock ${filepath}: ${error.message}`);
      }
    }
    
    this.locks.clear();
  }
}

const fileLockManager = new FileLockManager();
```

2. **Implement single-instance enforcement:**

```typescript
class SingleInstanceEnforcer {
  private lockfiles: Map<string, string> = new Map();
  
  async acquireInstanceLock(agentId: string): Promise<void> {
    const lockPath = path.join(
      pathManager.getCognitiveControllerDir(agentId),
      'instance.lock'
    );
    
    try {
      // Try to acquire lock
      await lockfile.lock(lockPath, {
        retries: 0, // Don't retry
        realpath: false
      });
      
      this.lockfiles.set(agentId, lockPath);
      
      logger.info(`Instance lock acquired for agent: ${agentId}`);
    } catch (error) {
      if (error.code === 'ELOCKED') {
        throw new Error(`Another Cognitive Controller instance is already running for agent: ${agentId}`);
      }
      throw error;
    }
  }
  
  async releaseInstanceLock(agentId: string): Promise<void> {
    const lockPath = this.lockfiles.get(agentId);
    
    if (lockPath) {
      try {
        await lockfile.unlock(lockPath);
        this.lockfiles.delete(agentId);
        logger.info(`Instance lock released for agent: ${agentId}`);
      } catch (error) {
        logger.error(`Failed to release instance lock: ${error.message}`);
      }
    }
  }
  
  async releaseAll(): Promise<void> {
    for (const agentId of this.lockfiles.keys()) {
      await this.releaseInstanceLock(agentId);
    }
  }
}

const singleInstanceEnforcer = new SingleInstanceEnforcer();
```

3. **Use locks for critical operations:**

```typescript
class LockedSnapshotStore {
  async saveSnapshot(snapshot: Snapshot): Promise<void> {
    const dbPath = pathManager.getDatabasePath(snapshot.handshake.snapshot_id.split('-')[0]);
    
    await fileLockManager.withLock(dbPath, async () => {
      // Save snapshot with lock held
      await sqliteStore.saveSnapshot(snapshot);
      
      logger.info(`Snapshot saved with lock: ${snapshot.handshake.snapshot_id}`);
    });
  }
  
  async updateSnapshot(snapshotId: string, updates: Partial<Snapshot>): Promise<void> {
    const agentId = snapshotId.split('-')[0];
    const dbPath = pathManager.getDatabasePath(agentId);
    
    await fileLockManager.withLock(dbPath, async () => {
      // Update snapshot with lock held
      await sqliteStore.updateSnapshot(snapshotId, updates);
      
      logger.info(`Snapshot updated with lock: ${snapshotId}`);
    });
  }
}
```

---

## Summary: Q57-Q64 Key Findings

### Security & Privacy
- ✅ Per-agent secret storage in auth-profiles.json
- ✅ OAuth token refresh with file locking
- ✅ Gateway authentication via token handshake
- ✅ Webhook security with hooks.token
- ⚠️ No automatic encryption at rest (must implement)
- ⚠️ No built-in PII redaction (must implement)
- ⚠️ No GDPR/CCPA compliance features (must implement)
- ✅ External-content safety boundaries available

### File System
- ✅ Clear workspace structure with required files
- ✅ Reserved filenames: BOOTSTRAP.md, HEARTBEAT.md
- ✅ Daily log pattern: YYYY-MM-DD.md
- ✅ Bootstrap limits: 20K per file, 150K total
- ⚠️ No workspace size limit (must implement quota)
- ✅ Real-time file watcher with 1.5s debounce
- ✅ File locking for critical operations
- ✅ Single-instance enforcement needed

### Critical Implementation Priorities

1. **Security (HIGH PRIORITY)**
   - Implement optional database encryption
   - Secure secret management
   - Gateway authentication
   - Webhook security

2. **Privacy & Compliance**
   - PII detection and redaction
   - GDPR compliance (export, delete, anonymize)
   - Retention policies
   - Secure backups

3. **File System Management**
   - Initialize workspace structure
   - File size validation
   - Workspace quota management
   - File watcher with debouncing

4. **Concurrency Control**
   - File locking for critical writes
   - Single-instance enforcement
   - Prevent concurrent access

---

**Next Steps:**
- Continue with Q65-Q100 as answers become available
- Implement security and encryption prototypes
- Test file locking and single-instance enforcement
- Validate workspace structure and quotas

