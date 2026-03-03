# OpenClaw Research Findings: Testing & Advanced Features (Q65-Q78)

**Date:** 2026-03-03  
**Source:** OpenClaw Documentation Database  
**Status:** Q65-Q78 Answered

---

## Testing & Development (Q65-Q67)

### Q65: Testing Infrastructure

**Answer:**

**Available Tests:**
- E2E smoke tests for Docker environments
- QR import smoke tests

**Mock Gateway:**
- No explicit standalone mock gateway mentioned
- "Synthetic" provider available for testing

### Implications for Cognitive Controller

⚠️ **Limited Test Infrastructure:**
- Must build our own test harness
- No mock gateway for isolated testing

✅ **Synthetic Provider:**
- Can use for testing without real LLM calls
- Useful for CI/CD

### Action Items

1. **Implement test harness:**

```typescript
import { describe, it, expect, beforeEach, afterEach } from 'vitest';

describe('Cognitive Controller', () => {
  let testAgentId: string;
  
  beforeEach(async () => {
    // Create test agent
    testAgentId = `test-agent-${Date.now()}`;
    await FileStructure.initializeWorkspace(testAgentId);
  });
  
  afterEach(async () => {
    // Cleanup test agent
    const ccDir = pathManager.getCognitiveControllerDir(testAgentId);
    await fs.rm(ccDir, { recursive: true, force: true });
  });
  
  describe('Snapshot Creation', () => {
    it('should create snapshot with valid structure', async () => {
      const context = {
        agentId: testAgentId,
        objective: 'Test objective',
        facts: ['Fact 1', 'Fact 2']
      };
      
      const snapshot = await snapshotAssembly.createSnapshot(context);
      
      expect(snapshot.handshake.snapshot_id).toBeDefined();
      expect(snapshot.objective_now).toBe('Test objective');
      expect(snapshot.recent_facts).toHaveLength(2);
    });
    
    it('should respect token budget', async () => {
      const context = {
        agentId: testAgentId,
        objective: 'Test objective',
        facts: Array(100).fill('Long fact content here')
      };
      
      const snapshot = await snapshotAssembly.createSnapshot(context);
      const tokens = tokenEstimator.estimateTokens(JSON.stringify(snapshot), 'gpt-4');
      
      expect(tokens).toBeLessThanOrEqual(500);
    });
  });
  
  describe('Fact Extraction', () => {
    it('should extract facts from text', async () => {
      const text = 'The user wants to build a web app. The deadline is 2026-03-15.';
      const facts = await factExtractor.extract(text);
      
      expect(facts.length).toBeGreaterThan(0);
      expect(facts.some(f => f.content.includes('web app'))).toBe(true);
    });
  });
  
  describe('SQLite Store', () => {
    it('should save and retrieve snapshot', async () => {
      const snapshot = createTestSnapshot(testAgentId);
      
      await sqliteStore.saveSnapshot(snapshot);
      const retrieved = await sqliteStore.getLatestSnapshot(testAgentId);
      
      expect(retrieved?.handshake.snapshot_id).toBe(snapshot.handshake.snapshot_id);
    });
    
    it('should handle concurrent writes', async () => {
      const snapshots = Array(10).fill(null).map((_, i) => 
        createTestSnapshot(testAgentId, `snapshot-${i}`)
      );
      
      // Save concurrently
      await Promise.all(snapshots.map(s => sqliteStore.saveSnapshot(s)));
      
      const count = await sqliteStore.getSnapshotCount(testAgentId);
      expect(count).toBe(10);
    });
  });
});

function createTestSnapshot(agentId: string, id?: string): Snapshot {
  return {
    handshake: {
      snapshot_id: id || generateUUID(),
      created_at: new Date().toISOString(),
      expires_at: new Date(Date.now() + 48 * 3600000).toISOString(),
      source_event_ids: [],
      schema_version: '1.0'
    },
    objective_now: 'Test objective',
    active_threads: [],
    recent_facts: [],
    hard_constraints: [],
    decisions: [],
    open_questions: [],
    tool_state: { available: [], unavailable: [], health: 'unknown' },
    last_actions: []
  };
}
```

2. **Use Synthetic provider for testing:**

```json5
// Test configuration
{
  "agents": {
    "list": [
      {
        "id": "test-agent",
        "model": {
          "provider": "synthetic",
          "model": "test-model"
        }
      }
    ]
  }
}
```

---

### Q66: Development Mode for Plugins

**Answer:**

**Skills:**
- Watchers pick up changes on next agent turn
- Default 250ms debounce

**Infrastructure Plugins:**
- Require manual Gateway restart for changes

**Debugging:**
- Verbose logging via `--verbose` flag
- npm notice-level logs and shell-level tracing

### Implications for Cognitive Controller

⚠️ **No Hot Reload:**
- Infrastructure changes require restart
- Slow development cycle

✅ **Skill Hot Reload:**
- Can develop skills with fast iteration
- Changes picked up automatically

### Action Items

1. **Enable development mode:**

```typescript
class DevelopmentMode {
  private isDevMode: boolean;
  
  constructor() {
    this.isDevMode = process.env.NODE_ENV === 'development';
  }
  
  async enableVerboseLogging(): Promise<void> {
    if (this.isDevMode) {
      logger.level = 'debug';
      logger.info('Development mode: verbose logging enabled');
    }
  }
  
  async watchForChanges(): Promise<void> {
    if (!this.isDevMode) return;
    
    logger.info('Development mode: watching for code changes');
    
    // Watch source files
    const watcher = chokidar.watch('src/**/*.ts', {
      persistent: true,
      ignoreInitial: true
    });
    
    watcher.on('change', (filepath) => {
      logger.warn(`Code changed: ${filepath} - restart required`);
      // Could trigger automatic restart here
    });
  }
  
  logPerformanceMetrics(): void {
    if (!this.isDevMode) return;
    
    setInterval(() => {
      const stats = perfMonitor.getStats('snapshot_creation');
      if (stats) {
        logger.debug(`Performance: snapshot_creation p50=${stats.p50}ms p95=${stats.p95}ms`);
      }
    }, 10000);
  }
}

const devMode = new DevelopmentMode();
```

---

### Q67: Plugin Debugging Experience

**Answer:**

**Primary Method:**
- Follow logs: `openclaw logs --follow`

**Web Control UI:**
- Real-time streaming of tool output
- Agent events show execution as it happens

### Implications for Cognitive Controller

✅ **Log Streaming:**
- Can follow logs in real-time
- Good for debugging

✅ **Control UI:**
- Visual monitoring of execution
- Useful for development

### Action Items

1. **Enhance logging for debugging:**

```typescript
class DebugLogger {
  logSnapshotCreation(snapshot: Snapshot, context: any): void {
    logger.debug('=== Snapshot Creation ===');
    logger.debug(`Snapshot ID: ${snapshot.handshake.snapshot_id}`);
    logger.debug(`Objective: ${snapshot.objective_now}`);
    logger.debug(`Active Threads: ${snapshot.active_threads.length}`);
    logger.debug(`Recent Facts: ${snapshot.recent_facts.length}`);
    logger.debug(`Constraints: ${snapshot.hard_constraints.length}`);
    logger.debug(`Context: ${JSON.stringify(context, null, 2)}`);
    
    // Log token usage
    const tokens = tokenEstimator.estimateTokens(JSON.stringify(snapshot), 'gpt-4');
    logger.debug(`Token usage: ${tokens} / 500`);
  }
  
  logFactExtraction(text: string, facts: AtomicFact[]): void {
    logger.debug('=== Fact Extraction ===');
    logger.debug(`Input length: ${text.length} characters`);
    logger.debug(`Extracted facts: ${facts.length}`);
    
    for (const fact of facts) {
      logger.debug(`  - ${fact.content} (confidence: ${fact.confidence})`);
    }
  }
  
  logPerformance(operation: string, duration: number): void {
    const target = PERFORMANCE_TARGETS[operation];
    const status = target && duration > target ? '⚠️ SLOW' : '✓';
    
    logger.debug(`${status} ${operation}: ${duration}ms${target ? ` (target: ${target}ms)` : ''}`);
  }
}

const debugLogger = new DebugLogger();
```

2. **Emit events for Control UI:**

```typescript
class ControlUIEmitter {
  async emitSnapshotCreated(snapshot: Snapshot): Promise<void> {
    await gateway.emit({
      type: 'indicator',
      subtype: 'cognitive_controller',
      status: 'snapshot_created',
      data: {
        snapshotId: snapshot.handshake.snapshot_id,
        objective: snapshot.objective_now,
        threadCount: snapshot.active_threads.length,
        factCount: snapshot.recent_facts.length
      }
    });
  }
  
  async emitHealthStatus(health: HealthStatus): Promise<void> {
    await gateway.emit({
      type: 'indicator',
      subtype: 'cognitive_controller',
      status: health.status,
      data: {
        agents: health.agents.length,
        totalSnapshots: health.metrics.totalSnapshots,
        totalFacts: health.metrics.totalFacts
      }
    });
  }
}

const controlUIEmitter = new ControlUIEmitter();
```

---

## Versioning & Compatibility (Q68-Q70)

### Q68: API Versioning

**Answer:**

**Versioning Scheme:**
- Format: `vYYYY.M.D` or `vYYYY.M.D-<patch>`
- Example: `v2026.3.3` or `v2026.3.3-1`

**Current State:**
- Pre-1.0 (fast-moving)
- Breaking changes possible

### Implications for Cognitive Controller

⚠️ **Pre-1.0 Instability:**
- OpenClaw API may change
- Must track version compatibility

✅ **Clear Versioning:**
- Easy to track releases
- Date-based makes sense

### Action Items

1. **Track OpenClaw version compatibility:**

```typescript
interface VersionCompatibility {
  minVersion: string;
  maxVersion?: string;
  breaking: string[];
}

class VersionManager {
  private readonly COMPATIBILITY: VersionCompatibility = {
    minVersion: 'v2026.1.1',
    maxVersion: undefined, // No max yet
    breaking: [
      'v2026.2.1', // Example breaking change
    ]
  };
  
  async checkCompatibility(): Promise<void> {
    const openClawVersion = await this.getOpenClawVersion();
    
    if (!this.isCompatible(openClawVersion)) {
      throw new Error(
        `Incompatible OpenClaw version: ${openClawVersion}. ` +
        `Required: >= ${this.COMPATIBILITY.minVersion}`
      );
    }
    
    if (this.hasBreakingChanges(openClawVersion)) {
      logger.warn(
        `OpenClaw version ${openClawVersion} includes breaking changes. ` +
        `Please review migration guide.`
      );
    }
    
    logger.info(`OpenClaw version ${openClawVersion} is compatible`);
  }
  
  private async getOpenClawVersion(): Promise<string> {
    // Get version from Gateway
    const response = await fetch('http://127.0.0.1:18789/api/version');
    const data = await response.json();
    return data.version;
  }
  
  private isCompatible(version: string): boolean {
    return this.compareVersions(version, this.COMPATIBILITY.minVersion) >= 0;
  }
  
  private hasBreakingChanges(version: string): boolean {
    return this.COMPATIBILITY.breaking.some(breaking => 
      this.compareVersions(version, breaking) >= 0
    );
  }
  
  private compareVersions(a: string, b: string): number {
    // Remove 'v' prefix
    const aParts = a.replace('v', '').split(/[.-]/).map(Number);
    const bParts = b.replace('v', '').split(/[.-]/).map(Number);
    
    for (let i = 0; i < Math.max(aParts.length, bParts.length); i++) {
      const aPart = aParts[i] || 0;
      const bPart = bParts[i] || 0;
      
      if (aPart > bPart) return 1;
      if (aPart < bPart) return -1;
    }
    
    return 0;
  }
}

const versionManager = new VersionManager();
```

---

### Q69: Communication of Breaking Changes

**Answer:**

**Method:**
- `openclaw doctor` handles breaking changes
- Explains legacy keys found
- Shows migrations applied to `openclaw.json`

### Implications for Cognitive Controller

✅ **Automated Migration:**
- OpenClaw handles its own migrations
- We should follow same pattern

✅ **Clear Communication:**
- Doctor explains changes
- Good UX for users

### Action Items

1. **Implement migration system:**

```typescript
interface Migration {
  version: string;
  description: string;
  migrate: (config: any) => any;
}

class MigrationManager {
  private migrations: Migration[] = [
    {
      version: 'v1.1.0',
      description: 'Rename snapshotBudget to snapshotTokenBudget',
      migrate: (config) => {
        if (config.snapshotBudget) {
          config.snapshotTokenBudget = config.snapshotBudget;
          delete config.snapshotBudget;
        }
        return config;
      }
    },
    {
      version: 'v1.2.0',
      description: 'Move database path to .cognitive-controller directory',
      migrate: (config) => {
        if (config.databasePath && !config.databasePath.includes('.cognitive-controller')) {
          config.databasePath = path.join('.cognitive-controller', 'memory.db');
        }
        return config;
      }
    }
  ];
  
  async migrateConfig(agentId: string, currentVersion: string): Promise<void> {
    logger.info(`Checking for migrations from ${currentVersion}`);
    
    const configPath = this.getConfigPath(agentId);
    let config = await this.loadConfig(configPath);
    
    let applied = 0;
    
    for (const migration of this.migrations) {
      if (this.shouldApplyMigration(currentVersion, migration.version)) {
        logger.info(`Applying migration: ${migration.description}`);
        config = migration.migrate(config);
        applied++;
      }
    }
    
    if (applied > 0) {
      await this.saveConfig(configPath, config);
      logger.info(`Applied ${applied} migrations`);
    } else {
      logger.info('No migrations needed');
    }
  }
  
  private shouldApplyMigration(currentVersion: string, migrationVersion: string): boolean {
    return versionManager.compareVersions(migrationVersion, currentVersion) > 0;
  }
  
  private getConfigPath(agentId: string): string {
    return path.join(
      pathManager.getCognitiveControllerDir(agentId),
      'config.json'
    );
  }
  
  private async loadConfig(configPath: string): Promise<any> {
    try {
      const content = await fs.readFile(configPath, 'utf-8');
      return JSON.parse(content);
    } catch (error) {
      if (error.code === 'ENOENT') {
        return {}; // Default config
      }
      throw error;
    }
  }
  
  private async saveConfig(configPath: string, config: any): Promise<void> {
    await fs.writeFile(configPath, JSON.stringify(config, null, 2), 'utf-8');
  }
}

const migrationManager = new MigrationManager();
```

---

### Q70: Backward Compatibility Policy

**Answer:**

**Automated Migrations:**
- Gateway auto-runs doctor migrations on startup
- Detects legacy config formats
- Migrates history, auth, models to current per-agent paths
- No manual intervention required

### Implications for Cognitive Controller

✅ **Automatic Migration:**
- Should auto-migrate on startup
- No user intervention needed

✅ **Per-Agent Paths:**
- Follow OpenClaw pattern
- Migrate to per-agent structure

### Action Items

1. **Auto-run migrations on startup:**

```typescript
class StartupMigration {
  async runStartupMigrations(): Promise<void> {
    logger.info('Running startup migrations...');
    
    const agents = await this.getAllAgents();
    
    for (const agentId of agents) {
      try {
        await this.migrateAgent(agentId);
      } catch (error) {
        logger.error(`Migration failed for agent ${agentId}: ${error.message}`);
      }
    }
    
    logger.info('Startup migrations complete');
  }
  
  private async migrateAgent(agentId: string): Promise<void> {
    // Check current version
    const currentVersion = await this.getCurrentVersion(agentId);
    
    if (!currentVersion) {
      logger.info(`No version found for agent ${agentId}, assuming fresh install`);
      await this.setVersion(agentId, CURRENT_VERSION);
      return;
    }
    
    if (currentVersion === CURRENT_VERSION) {
      logger.debug(`Agent ${agentId} already at current version`);
      return;
    }
    
    logger.info(`Migrating agent ${agentId} from ${currentVersion} to ${CURRENT_VERSION}`);
    
    // Run migrations
    await migrationManager.migrateConfig(agentId, currentVersion);
    await this.migrateDatabaseSchema(agentId, currentVersion);
    await this.migrateFileStructure(agentId, currentVersion);
    
    // Update version
    await this.setVersion(agentId, CURRENT_VERSION);
    
    logger.info(`Agent ${agentId} migrated successfully`);
  }
  
  private async getCurrentVersion(agentId: string): Promise<string | null> {
    const versionPath = path.join(
      pathManager.getCognitiveControllerDir(agentId),
      'version.txt'
    );
    
    try {
      return await fs.readFile(versionPath, 'utf-8');
    } catch (error) {
      if (error.code === 'ENOENT') return null;
      throw error;
    }
  }
  
  private async setVersion(agentId: string, version: string): Promise<void> {
    const versionPath = path.join(
      pathManager.getCognitiveControllerDir(agentId),
      'version.txt'
    );
    
    await fs.writeFile(versionPath, version, 'utf-8');
  }
  
  private async migrateDatabaseSchema(agentId: string, fromVersion: string): Promise<void> {
    // Migrate database schema if needed
    logger.info(`Migrating database schema for agent ${agentId}`);
    await sqliteStore.migrateSchema(agentId, fromVersion);
  }
  
  private async migrateFileStructure(agentId: string, fromVersion: string): Promise<void> {
    // Migrate file structure if needed
    logger.info(`Migrating file structure for agent ${agentId}`);
    
    // Example: Move old database location to new location
    const oldDbPath = path.join(pathManager.getWorkspace(agentId), 'memory.db');
    const newDbPath = pathManager.getDatabasePath(agentId);
    
    try {
      await fs.access(oldDbPath);
      await fs.rename(oldDbPath, newDbPath);
      logger.info(`Moved database from ${oldDbPath} to ${newDbPath}`);
    } catch (error) {
      if (error.code !== 'ENOENT') throw error;
    }
  }
  
  private async getAllAgents(): Promise<string[]> {
    const config = await gateway.getConfig();
    return config.agents.list
      .filter(a => a.skills?.['cognitive-controller']?.enabled)
      .map(a => a.id);
  }
}

const startupMigration = new StartupMigration();

// Run on plugin initialization
export async function initialize(gateway: Gateway): Promise<void> {
  await startupMigration.runStartupMigrations();
  // ... rest of initialization
}
```

---

## Observability & Monitoring (Q71-Q74)

### Q71: openclaw doctor Diagnostic Tool

**Answer:**

**Primary Functions:**
- Configuration: Normalizes legacy values, migrates deprecated keys
- State Integrity: Verifies permissions, writability of session directories/transcripts
- Connectivity: Checks Gateway health, port collisions (18789), model auth health
- Service Audit: Repairs supervisor configs (launchd/systemd/schtasks)

### Implications for Cognitive Controller

✅ **Diagnostic Pattern:**
- Should implement similar doctor command
- Check configuration, state, connectivity

✅ **Auto-Repair:**
- Fix common issues automatically
- Good UX

### Action Items

1. **Implement doctor command:**

```typescript
class CognitiveControllerDoctor {
  async runDiagnostics(agentId?: string): Promise<DiagnosticReport> {
    logger.info('Running Cognitive Controller diagnostics...');
    
    const agents = agentId ? [agentId] : await this.getAllAgents();
    const results: AgentDiagnostic[] = [];
    
    for (const id of agents) {
      results.push(await this.diagnoseAgent(id));
    }
    
    const report: DiagnosticReport = {
      timestamp: new Date().toISOString(),
      agents: results,
      summary: this.summarize(results)
    };
    
    // Print report
    this.printReport(report);
    
    return report;
  }
  
  private async diagnoseAgent(agentId: string): Promise<AgentDiagnostic> {
    const checks: DiagnosticCheck[] = [];
    
    // Check 1: Configuration
    checks.push(await this.checkConfiguration(agentId));
    
    // Check 2: Database
    checks.push(await this.checkDatabase(agentId));
    
    // Check 3: File Structure
    checks.push(await this.checkFileStructure(agentId));
    
    // Check 4: Permissions
    checks.push(await this.checkPermissions(agentId));
    
    // Check 5: Connectivity
    checks.push(await this.checkConnectivity(agentId));
    
    // Check 6: Health
    checks.push(await this.checkHealth(agentId));
    
    const issues = checks.filter(c => !c.passed);
    const autoFixed = await this.autoFix(agentId, issues);
    
    return {
      agentId,
      healthy: issues.length === 0,
      checks,
      issues: issues.map(c => c.message),
      autoFixed
    };
  }
  
  private async checkConfiguration(agentId: string): Promise<DiagnosticCheck> {
    try {
      const configPath = path.join(
        pathManager.getCognitiveControllerDir(agentId),
        'config.json'
      );
      
      await fs.access(configPath);
      const content = await fs.readFile(configPath, 'utf-8');
      JSON.parse(content); // Validate JSON
      
      return {
        name: 'Configuration',
        passed: true,
        message: 'Configuration file valid'
      };
    } catch (error) {
      return {
        name: 'Configuration',
        passed: false,
        message: `Configuration issue: ${error.message}`,
        fixable: true
      };
    }
  }
  
  private async checkDatabase(agentId: string): Promise<DiagnosticCheck> {
    try {
      const dbPath = pathManager.getDatabasePath(agentId);
      await fs.access(dbPath);
      
      const healthy = await sqliteStore.checkIntegrity(agentId);
      
      if (!healthy) {
        return {
          name: 'Database',
          passed: false,
          message: 'Database corrupted',
          fixable: true
        };
      }
      
      return {
        name: 'Database',
        passed: true,
        message: 'Database healthy'
      };
    } catch (error) {
      return {
        name: 'Database',
        passed: false,
        message: `Database missing: ${error.message}`,
        fixable: true
      };
    }
  }
  
  private async checkFileStructure(agentId: string): Promise<DiagnosticCheck> {
    const workspace = pathManager.getWorkspace(agentId);
    const requiredDirs = Object.values(FileStructure.DIRECTORIES);
    
    const missing: string[] = [];
    
    for (const dir of requiredDirs) {
      const dirPath = path.join(workspace, dir);
      try {
        await fs.access(dirPath);
      } catch (error) {
        if (error.code === 'ENOENT') {
          missing.push(dir);
        }
      }
    }
    
    if (missing.length > 0) {
      return {
        name: 'File Structure',
        passed: false,
        message: `Missing directories: ${missing.join(', ')}`,
        fixable: true
      };
    }
    
    return {
      name: 'File Structure',
      passed: true,
      message: 'File structure complete'
    };
  }
  
  private async checkPermissions(agentId: string): Promise<DiagnosticCheck> {
    const ccDir = pathManager.getCognitiveControllerDir(agentId);
    
    try {
      // Test write permission
      const testFile = path.join(ccDir, '.permission-test');
      await fs.writeFile(testFile, 'test', 'utf-8');
      await fs.unlink(testFile);
      
      return {
        name: 'Permissions',
        passed: true,
        message: 'Permissions OK'
      };
    } catch (error) {
      return {
        name: 'Permissions',
        passed: false,
        message: `Permission denied: ${error.message}`,
        fixable: false
      };
    }
  }
  
  private async checkConnectivity(agentId: string): Promise<DiagnosticCheck> {
    try {
      const response = await fetch('http://127.0.0.1:18789/api/health', {
        timeout: 5000
      });
      
      if (response.ok) {
        return {
          name: 'Connectivity',
          passed: true,
          message: 'Gateway reachable'
        };
      } else {
        return {
          name: 'Connectivity',
          passed: false,
          message: `Gateway returned ${response.status}`,
          fixable: false
        };
      }
    } catch (error) {
      return {
        name: 'Connectivity',
        passed: false,
        message: `Cannot reach Gateway: ${error.message}`,
        fixable: false
      };
    }
  }
  
  private async checkHealth(agentId: string): Promise<DiagnosticCheck> {
    try {
      const health = await cognitiveControllerHealth.checkAgentHealth(agentId);
      
      if (health.status === 'healthy') {
        return {
          name: 'Health',
          passed: true,
          message: 'Agent healthy'
        };
      } else {
        return {
          name: 'Health',
          passed: false,
          message: `Agent ${health.status}: ${health.error || 'unknown issue'}`,
          fixable: true
        };
      }
    } catch (error) {
      return {
        name: 'Health',
        passed: false,
        message: `Health check failed: ${error.message}`,
        fixable: false
      };
    }
  }
  
  private async autoFix(agentId: string, issues: DiagnosticCheck[]): Promise<string[]> {
    const fixed: string[] = [];
    
    for (const issue of issues) {
      if (!issue.fixable) continue;
      
      try {
        if (issue.name === 'Configuration') {
          await this.fixConfiguration(agentId);
          fixed.push('Configuration');
        } else if (issue.name === 'Database') {
          await stateRecovery.repairDatabase(agentId);
          fixed.push('Database');
        } else if (issue.name === 'File Structure') {
          await FileStructure.initializeWorkspace(agentId);
          fixed.push('File Structure');
        } else if (issue.name === 'Health') {
          await stateRecovery.recoverFromCrash(agentId);
          fixed.push('Health');
        }
      } catch (error) {
        logger.error(`Failed to fix ${issue.name}: ${error.message}`);
      }
    }
    
    return fixed;
  }
  
  private async fixConfiguration(agentId: string): Promise<void> {
    const configPath = path.join(
      pathManager.getCognitiveControllerDir(agentId),
      'config.json'
    );
    
    const defaultConfig = {
      snapshotTokenBudget: 500,
      maxActiveThreads: 7,
      retentionDays: 90
    };
    
    await fs.writeFile(configPath, JSON.stringify(defaultConfig, null, 2), 'utf-8');
  }
  
  private printReport(report: DiagnosticReport): void {
    console.log('\n=== Cognitive Controller Diagnostics ===\n');
    console.log(`Timestamp: ${report.timestamp}`);
    console.log(`Summary: ${report.summary}\n`);
    
    for (const agent of report.agents) {
      console.log(`Agent: ${agent.agentId}`);
      console.log(`Status: ${agent.healthy ? '✓ Healthy' : '✗ Issues Found'}\n`);
      
      for (const check of agent.checks) {
        const icon = check.passed ? '✓' : '✗';
        console.log(`  ${icon} ${check.name}: ${check.message}`);
      }
      
      if (agent.autoFixed.length > 0) {
        console.log(`\n  Auto-fixed: ${agent.autoFixed.join(', ')}`);
      }
      
      console.log('');
    }
  }
  
  private summarize(results: AgentDiagnostic[]): string {
    const healthy = results.filter(r => r.healthy).length;
    const total = results.length;
    
    if (healthy === total) {
      return `All ${total} agents healthy`;
    } else {
      return `${healthy}/${total} agents healthy, ${total - healthy} need attention`;
    }
  }
  
  private async getAllAgents(): Promise<string[]> {
    const config = await gateway.getConfig();
    return config.agents.list
      .filter(a => a.skills?.['cognitive-controller']?.enabled)
      .map(a => a.id);
  }
}

interface DiagnosticCheck {
  name: string;
  passed: boolean;
  message: string;
  fixable?: boolean;
}

const cognitiveControllerDoctor = new CognitiveControllerDoctor();
```

---

### Q72: Web Control UI

**Answer:**

**Location:**
- Served at `http://127.0.0.1:18789/`

**Features:**
- Agent Management: View agent list and health status
- Session Viewer: Access chat history and session metadata
- Configuration: Interactive wizard and raw JSON editor
- Canvas: Host agent-editable HTML/CSS/JS for mobile node displays

### Implications for Cognitive Controller

✅ **Integration Point:**
- Can add Cognitive Controller UI to Control UI
- Show snapshots, facts, health

✅ **Real-time Updates:**
- Can stream events to UI
- Good for monitoring

### Action Items

1. **Add Cognitive Controller UI panel:**

```typescript
class ControlUIIntegration {
  async registerUIPanel(): Promise<void> {
    // Register custom UI panel with Gateway
    await gateway.registerUIPanel({
      id: 'cognitive-controller',
      title: 'Cognitive Controller',
      icon: 'brain',
      path: '/cognitive-controller',
      handler: this.renderPanel.bind(this)
    });
    
    logger.info('Control UI panel registered');
  }
  
  private async renderPanel(req: any, res: any): Promise<void> {
    const agents = await this.getAllAgents();
    const agentData = await Promise.all(
      agents.map(agentId => this.getAgentData(agentId))
    );
    
    const html = this.generateHTML(agentData);
    res.send(html);
  }
  
  private async getAgentData(agentId: string): Promise<any> {
    const health = await cognitiveControllerHealth.checkAgentHealth(agentId);
    const latestSnapshot = await sqliteStore.getLatestSnapshot(agentId);
    const recentFacts = await sqliteStore.getRecentFacts(agentId, 10);
    
    return {
      agentId,
      health,
      latestSnapshot,
      recentFacts
    };
  }
  
  private generateHTML(agentData: any[]): string {
    return `
<!DOCTYPE html>
<html>
<head>
  <title>Cognitive Controller</title>
  <style>
    body { font-family: system-ui; padding: 20px; }
    .agent { border: 1px solid #ccc; padding: 15px; margin: 10px 0; border-radius: 5px; }
    .healthy { border-color: #4caf50; }
    .degraded { border-color: #ff9800; }
    .unhealthy { border-color: #f44336; }
    .snapshot { background: #f5f5f5; padding: 10px; margin: 10px 0; border-radius: 3px; }
    .fact { padding: 5px; margin: 5px 0; background: #e3f2fd; border-radius: 3px; }
  </style>
</head>
<body>
  <h1>Cognitive Controller</h1>
  
  ${agentData.map(agent => `
    <div class="agent ${agent.health.status}">
      <h2>${agent.agentId}</h2>
      <p>Status: ${agent.health.status}</p>
      <p>Snapshots: ${agent.health.snapshotCount}</p>
      <p>Facts: ${agent.health.factCount}</p>
      
      ${agent.latestSnapshot ? `
        <div class="snapshot">
          <h3>Latest Snapshot</h3>
          <p><strong>ID:</strong> ${agent.latestSnapshot.handshake.snapshot_id}</p>
          <p><strong>Created:</strong> ${agent.latestSnapshot.handshake.created_at}</p>
          <p><strong>Objective:</strong> ${agent.latestSnapshot.objective_now}</p>
          <p><strong>Active Threads:</strong> ${agent.latestSnapshot.active_threads.length}</p>
        </div>
      ` : '<p>No snapshots yet</p>'}
      
      <h3>Recent Facts</h3>
      ${agent.recentFacts.map(fact => `
        <div class="fact">${fact.content}</div>
      `).join('')}
    </div>
  `).join('')}
  
  <script>
    // Auto-refresh every 10 seconds
    setTimeout(() => location.reload(), 10000);
  </script>
</body>
</html>
    `;
  }
  
  private async getAllAgents(): Promise<string[]> {
    const config = await gateway.getConfig();
    return config.agents.list
      .filter(a => a.skills?.['cognitive-controller']?.enabled)
      .map(a => a.id);
  }
}

const controlUIIntegration = new ControlUIIntegration();
```

---

### Q73: Health Check Endpoints

**Answer:**

**WebSocket API:**
- `health` command over WebSocket

**CLI:**
- `openclaw health`

**Docker:**
- `node dist/index.js health --token "$OPENCLAW_GATEWAY_TOKEN"`

### Implications for Cognitive Controller

✅ **Health Integration:**
- Can expose health via same mechanisms
- Standard pattern

### Action Items

1. **Expose health endpoint:**

```typescript
class HealthEndpoint {
  async registerHealthEndpoint(): Promise<void> {
    // Register with Gateway
    gateway.registerHealthProvider('cognitive-controller', async () => {
      return await cognitiveControllerHealth.getHealthStatus();
    });
    
    logger.info('Health endpoint registered');
  }
  
  async handleHealthCheck(req: any, res: any): Promise<void> {
    try {
      const health = await cognitiveControllerHealth.getHealthStatus();
      
      res.status(health.status === 'healthy' ? 200 : 503).json(health);
    } catch (error) {
      res.status(500).json({
        status: 'unhealthy',
        error: error.message
      });
    }
  }
}

const healthEndpoint = new HealthEndpoint();
```

---

### Q74: Heartbeat Mechanism

**Answer:**

**Frequency:**
- Default: 30 minutes
- Some auth types: 1 hour

**Payload:**
- Sent as user message
- Instructs agent to read `HEARTBEAT.md`
- Reply `HEARTBEAT_OK` if no attention needed

**Missed Heartbeats:**
- Skipped if main queue busy
- Retried later

### Implications for Cognitive Controller

✅ **Heartbeat Pattern:**
- Can use for periodic health checks
- Good for detecting issues

⚠️ **Queue Dependency:**
- Heartbeat skipped if busy
- Not guaranteed delivery

### Action Items

1. **Implement heartbeat monitoring:**

```typescript
class HeartbeatMonitor {
  private intervals: Map<string, NodeJS.Timeout> = new Map();
  private readonly HEARTBEAT_INTERVAL = 30 * 60 * 1000; // 30 minutes
  
  startMonitoring(agentId: string): void {
    if (this.intervals.has(agentId)) {
      return; // Already monitoring
    }
    
    const interval = setInterval(() => {
      this.sendHeartbeat(agentId);
    }, this.HEARTBEAT_INTERVAL);
    
    this.intervals.set(agentId, interval);
    
    logger.info(`Heartbeat monitoring started for agent: ${agentId}`);
  }
  
  stopMonitoring(agentId: string): void {
    const interval = this.intervals.get(agentId);
    
    if (interval) {
      clearInterval(interval);
      this.intervals.delete(agentId);
      logger.info(`Heartbeat monitoring stopped for agent: ${agentId}`);
    }
  }
  
  private async sendHeartbeat(agentId: string): Promise<void> {
    try {
      logger.debug(`Sending heartbeat for agent: ${agentId}`);
      
      // Check health
      const health = await cognitiveControllerHealth.checkAgentHealth(agentId);
      
      if (health.status !== 'healthy') {
        logger.warn(`Heartbeat detected unhealthy agent: ${agentId}`);
        await this.handleUnhealthyAgent(agentId, health);
      } else {
        logger.debug(`Heartbeat OK for agent: ${agentId}`);
      }
      
      // Record heartbeat
      await this.recordHeartbeat(agentId, health.status);
    } catch (error) {
      logger.error(`Heartbeat failed for agent ${agentId}: ${error.message}`);
    }
  }
  
  private async handleUnhealthyAgent(agentId: string, health: any): Promise<void> {
    // Attempt recovery
    logger.info(`Attempting recovery for unhealthy agent: ${agentId}`);
    
    try {
      await stateRecovery.recoverFromCrash(agentId);
      logger.info(`Recovery successful for agent: ${agentId}`);
    } catch (error) {
      logger.error(`Recovery failed for agent ${agentId}: ${error.message}`);
    }
  }
  
  private async recordHeartbeat(agentId: string, status: string): Promise<void> {
    await auditLogger.logEvent(agentId, {
      timestamp: new Date().toISOString(),
      agentId,
      eventType: 'heartbeat',
      details: { status }
    });
  }
  
  stopAll(): void {
    for (const agentId of this.intervals.keys()) {
      this.stopMonitoring(agentId);
    }
  }
}

const heartbeatMonitor = new HeartbeatMonitor();
```

---

## Advanced Features (Q75-Q78)

### Q75: QMD Backend

**Answer:**

**Architecture:**
- Experimental local-first search sidecar
- Swaps built-in SQLite indexer
- Runs as separate process via Bun and node-llama-cpp

**Features:**
- Combines BM25, vectors, and reranking
- High-performance retrieval

### Implications for Cognitive Controller

✅ **Alternative Backend:**
- Could use QMD for fact retrieval
- Better performance than SQLite

⚠️ **Experimental:**
- May not be stable
- Requires separate process

### Action Items

1. **Support QMD backend (optional):**

```typescript
class QMDBackend {
  private qmdProcess: ChildProcess | null = null;
  
  async start(): Promise<void> {
    logger.info('Starting QMD backend...');
    
    this.qmdProcess = spawn('bun', ['run', 'qmd-server.js'], {
      stdio: 'pipe'
    });
    
    this.qmdProcess.stdout?.on('data', (data) => {
      logger.debug(`QMD: ${data.toString()}`);
    });
    
    this.qmdProcess.stderr?.on('data', (data) => {
      logger.error(`QMD error: ${data.toString()}`);
    });
    
    // Wait for startup
    await this.waitForReady();
    
    logger.info('QMD backend started');
  }
  
  async search(query: string, options: any): Promise<any[]> {
    const response = await fetch('http://localhost:8765/search', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ query, ...options })
    });
    
    return await response.json();
  }
  
  async stop(): Promise<void> {
    if (this.qmdProcess) {
      this.qmdProcess.kill();
      this.qmdProcess = null;
      logger.info('QMD backend stopped');
    }
  }
  
  private async waitForReady(): Promise<void> {
    const maxAttempts = 30;
    
    for (let i = 0; i < maxAttempts; i++) {
      try {
        const response = await fetch('http://localhost:8765/health');
        if (response.ok) return;
      } catch (error) {
        // Not ready yet
      }
      
      await new Promise(resolve => setTimeout(resolve, 1000));
    }
    
    throw new Error('QMD backend failed to start');
  }
}
```

---

### Q76: "Agent Send" Capability

**Answer:**

**Purpose:**
- Coordination between agents

**Isolation:**
- Isolated by default
- Must explicitly enable via `tools.agentToAgent`

**Retry Policy:**
- Sub-agent announcements: up to 3 retries
- Handles busy requester sessions

### Implications for Cognitive Controller

✅ **Multi-Agent Support:**
- Can share memory between agents (Phase 4+)
- Useful for coordination

⚠️ **Must Enable:**
- Not available by default
- Requires configuration

### Action Items

Already implemented in Q39 (Agent Communication section).

---

### Q77: Shared Skills

**Answer:**

**Locations:**
- Bundled: With install
- Managed/local: `~/.openclaw/skills`
- Workspace-specific: `<workspace>/skills`

**Shared Skills:**
- Available to all agents from `~/.openclaw/skills`

### Implications for Cognitive Controller

✅ **Deployment Model:**
- Deploy as shared skill in `~/.openclaw/skills`
- All agents get access

Already covered in Q41 (Shared Resources section).

---

### Q78: Cron Job System

**Answer:**

**Format:**
- 5 or 6-field cron expressions
- Uses `croner` library

**Execution:**
- Main session: Enqueues system events
- Isolated session: Starts fresh agent turn

**Guarantees:**
- Top-of-hour jobs staggered by up to 5 minutes
- Prevents load spikes

### Implications for Cognitive Controller

✅ **Scheduled Tasks:**
- Can use for periodic snapshot creation
- Roll-up generation
- Cleanup tasks

✅ **Load Management:**
- Automatic staggering prevents spikes

### Action Items

1. **Register cron jobs:**

```typescript
class CronJobManager {
  async registerJobs(): Promise<void> {
    // Hourly snapshot creation
    await gateway.registerCronJob({
      id: 'cc-hourly-snapshot',
      schedule: '0 * * * *', // Every hour
      handler: async (agentId) => {
        await this.createScheduledSnapshot(agentId);
      }
    });
    
    // Daily roll-up
    await gateway.registerCronJob({
      id: 'cc-daily-rollup',
      schedule: '0 2 * * *', // 2 AM daily
      handler: async (agentId) => {
        await rollUpEngine.triggerRollUp('scheduled');
      }
    });
    
    // Weekly cleanup
    await gateway.registerCronJob({
      id: 'cc-weekly-cleanup',
      schedule: '0 3 * * 0', // 3 AM Sunday
      handler: async (agentId) => {
        await retentionPolicy.enforceRetention(agentId);
        await quotaManager.enforceQuota(agentId);
      }
    });
    
    logger.info('Cron jobs registered');
  }
  
  private async createScheduledSnapshot(agentId: string): Promise<void> {
    logger.info(`Creating scheduled snapshot for agent: ${agentId}`);
    
    try {
      const context = await this.gatherContext(agentId);
      const snapshot = await snapshotAssembly.createSnapshot(context);
      
      await sqliteStore.saveSnapshot(snapshot);
      
      logger.info(`Scheduled snapshot created: ${snapshot.handshake.snapshot_id}`);
    } catch (error) {
      logger.error(`Scheduled snapshot failed: ${error.message}`);
    }
  }
  
  private async gatherContext(agentId: string): Promise<any> {
    // Gather context for snapshot
    return {
      agentId,
      trigger: 'scheduled',
      timestamp: new Date().toISOString()
    };
  }
}

const cronJobManager = new CronJobManager();
```

---

## Summary: Q65-Q78 Key Findings

### Testing & Development
- ⚠️ Limited test infrastructure (must build own)
- ✅ Synthetic provider for testing
- ⚠️ No hot reload for infrastructure plugins (restart required)
- ✅ Skills hot reload with 250ms debounce
- ✅ Verbose logging for debugging
- ✅ Control UI for real-time monitoring

### Versioning & Compatibility
- ✅ Clear versioning: vYYYY.M.D
- ⚠️ Pre-1.0 (fast-moving, breaking changes possible)
- ✅ openclaw doctor handles migrations
- ✅ Automated migrations on startup
- ✅ Backward compatibility maintained

### Observability & Monitoring
- ✅ openclaw doctor for diagnostics and repair
- ✅ Web Control UI at http://127.0.0.1:18789/
- ✅ Health check endpoints (WebSocket, CLI, Docker)
- ✅ Heartbeat mechanism (30 min default)
- ✅ Real-time event streaming

### Advanced Features
- ✅ QMD backend (experimental, high-performance)
- ✅ Agent Send for multi-agent coordination
- ✅ Shared skills from ~/.openclaw/skills
- ✅ Cron jobs with automatic staggering

### Critical Implementation Priorities

1. **Testing Infrastructure**
   - Build test harness
   - Use Synthetic provider
   - Implement E2E tests

2. **Diagnostics & Monitoring**
   - Implement doctor command
   - Register health checks
   - Add Control UI panel
   - Heartbeat monitoring

3. **Version Management**
   - Track OpenClaw compatibility
   - Implement migration system
   - Auto-run migrations on startup

4. **Scheduled Tasks**
   - Register cron jobs for snapshots
   - Daily roll-ups
   - Weekly cleanup

---

**Research Complete: 78/100 Questions Answered**

**Next Steps:**
- Continue with Q79-Q100 as answers become available
- Update requirements.md with all findings
- Begin Phase 0 implementation

