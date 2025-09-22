# BHoM CI Workflow Diagrams

This file contains all the diagrams explaining the BHoM Build Workflow. Each section represents a separate tab/diagram that can be imported into Draw.io or other diagramming tools.

---

## Tab 1: Main Overview - BHoM Build Workflow

```mermaid
graph TD
    A["`**🚀 BHoM Build Workflow**
    
    **Triggers:** workflow_call
    **Inputs:** repository_name, branch_name, dotnet_version
    **Concurrency:** Cancel in-progress runs on new commits`"] --> B["`**📋 Job 1: setup-configurations-and-dependencies**
    **Purpose:** Setup configs and download available dependencies
    **Duration:** ~2-3 minutes
    **Runs on:** ubuntu-latest
    
    📋 **See Detailed Diagram 2**`"]
    
    B --> C{"`**Missing dependencies?**`"}
    C -->|Yes| D["`**🔨 Job 2: build-dependencies-with-no-artifacts**
    **Purpose:** Build missing dependencies only
    **Duration:** ~5-15 minutes (conditional)
    **Runs on:** windows-latest
    
    📋 **See Detailed Diagram 3**`"]
    C -->|No| E["`**Skip dependency building**`"]
    
    D --> F["`**🔨 Job 3: build (Matrix)**
    **Purpose:** Build project for each configuration
    **Matrix:** build-configuration (Debug, Release, etc.)
    **Duration:** ~3-10 minutes per config
    **Runs on:** windows-latest
    
    📋 **See Detailed Diagram 4**`"]
    E --> F
    
    F --> G["`**🧪 Job 4: tests**
    **Purpose:** Run tests and generate reports
    **Duration:** ~2-5 minutes
    **Runs on:** windows-latest
    
    📋 **See Detailed Diagram 5**`"]
    
    B -.->|"`**Artifacts & Outputs:**
    • build-configurations
    • release-configurations  
    • dependencies-built (complete/incomplete)
    • changed-files
    • missing-dependencies
    • branch-map
    • Dependencies or Dependencies-Incomplete`"| C
    
    D -.->|"`**Artifacts & Outputs:**
    • dependencies-built (true/false)
    • Dependencies (complete set)`"| F
    
    F -.->|"`**Artifacts & Outputs:**
    • build-outputs-$branch-$config
    • bhom-assemblies-$config
    • updated-assemblies-$config.txt`"| G
    
    H["`**🔍 Key Features:**
    
    **Cross-Platform Optimization:** Ubuntu for config discovery
    **Conditional Building:** Only build missing dependencies
    **Priority-Based Downloads:** Matching configs → Release fallback
    **Path Handling:** Ubuntu→Windows compatibility
    
    **⚠️ Critical Error Conditions:**
    • Build failures → EXIT 1  
    • No test DLLs found → EXIT 1
    • Test failures → EXIT 1`"]
    
    I["`**📈 Performance Optimizations:**
    
    • **Ubuntu first job** - faster, cheaper for configs
    • **Conditional Windows job** - only when needed
    • **Artifact priority system** - matching configs first
    • **Path conversion** - cross-platform compatibility
    • **Parallel artifact checking** while workflows run
    • **Matrix builds** for multiple configurations`"]
    
    J["`**🔧 Authentication & Access:**
    
    • **GitHub App authentication** (preferred)
    • **Personal Access Token** fallback
    • **Default GITHUB_TOKEN** as last resort
    • **Cross-repository access** for dependencies`"]
    
    style A fill:#e3f2fd
    style B fill:#e8f5e8
    style C fill:#ffecb3
    style D fill:#f3e5f5
    style E fill:#e0f2f1
    style F fill:#fff3e0
    style G fill:#fce4ec
    style H fill:#f1f8e9
    style I fill:#e0f2f1
    style J fill:#f1f8e9
```

---

## Tab 2: setup-configurations-and-dependencies Job Detail

```mermaid
graph TD
    A["`**setup-configurations-and-dependencies Job**
    Runs on: ubuntu-latest`"] --> B["`**Discover Build Configurations**
    • Parse .sln file (bash)
    • Extract build configs
    • Identify Release configs
    • Output JSON arrays`"]
    
    B --> C["`**Parse Dependencies**
    • Read dependencies.txt (bash)
    • Extract dependency list
    • Convert to JSON`"]
    
    C --> D["`**Get Changed Files**
    Compare with develop branch`"]
    
    D --> E{"`**Branch = develop?**`"}
    E -->|Yes| F["`**No changed files tracking**`"]
    E -->|No| G["`**Git diff vs develop (bash)**
    • Fetch develop branch
    • Compare files
    • Generate RELATIVE paths (cross-platform)`"]
    
    F --> H["`**Create BHoM folder**
    mkdir -p /tmp/BHoM/Assemblies`"]
    G --> H
    
    H --> I["`**Setup Authentication**
    • Check GitHub App secrets
    • Generate token if available`"]
    
    I --> J["`**Determine Dependency Branches**
    For each dependency:`"]
    J --> K{"`**Target branch exists?**`"}
    K -->|Yes| L["`**Use target branch**`"]
    K -->|No| M["`**Use develop branch**`"]
    
    L --> N["`**Download Dependency Artifacts**
    Priority-based download system`"]
    M --> N
    
    N --> O{"`**All dependencies found?**`"}
    O -->|Yes| P["`**Mark as 'complete'**
    dependencies-built=complete`"]
    O -->|No| Q["`**Mark as 'incomplete'**
    dependencies-built=incomplete
    Track missing dependencies`"]
    
    P --> R["`**Upload Dependencies artifact**
    Complete dependency set`"]
    Q --> S["`**Upload Dependencies-Incomplete artifact**
    Partial dependency set + missing list`"]
    
    R --> T["`**Job Complete**
    Second job will be SKIPPED`"]
    S --> U["`**Job Complete**
    Second job will RUN`"]
    
    style A fill:#e8f5e8
    style E fill:#ffecb3
    style K fill:#ffecb3
    style O fill:#ffecb3
    style P fill:#c8e6c9
    style Q fill:#ffcdd2
```

---

## Tab 3: build-dependencies-with-no-artifacts Job Detail

```mermaid
graph TD
    A["`**build-dependencies-with-no-artifacts Job**
    Runs on: windows-latest
    Condition: IF dependencies-built == 'incomplete'`"] --> B["`**Setup Environment**
    • Checkout code
    • Setup MSBuild & NuGet
    • Create BHoM folder`"]
    
    B --> C["`**Download Dependencies-Incomplete**
    Download partial dependency set`"]
    
    C --> D["`**Restore Existing Dependencies**
    Copy to BHoM folder`"]
    
    D --> E["`**Get Missing Dependencies List**
    Read missing-dependencies.json`"]
    
    E --> F{"`**Any missing dependencies?**`"}
    F -->|No| G["`**Job Complete - Nothing to build**`"]
    F -->|Yes| H["`**Setup Authentication**
    Generate GitHub App token`"]
    
    H --> I["`**Clone Missing Dependencies**
    For each missing dependency:`"]
    I --> J["`**Clone Repository**
    • Get branch from branch-map
    • Clone to Repositories folder
    • Checkout correct branch`"]
    
    J --> K{"`**.sln file exists?**`"}
    K -->|Yes| L["`**Mark as valid for building**`"]
    K -->|No| M["`**Skip - no solution file**`"]
    
    L --> N["`**Analyze Dependency Order**
    • Read dependencies.txt files
    • Create dependency map
    • Topological sort
    • Generate build order`"]
    M --> N
    
    N --> O["`**Build Dependencies in Order**
    For each repository:`"]
    O --> P["`**Detect Project Type**
    • Check for packages.config (legacy)
    • Check for SDK-style projects`"]
    
    P --> Q{"`**Legacy project?**`"}
    Q -->|Yes| R["`**NuGet + MSBuild**
    • nuget restore
    • msbuild /restore /m /p:Configuration=Release`"]
    Q -->|No| S["`**Dotnet CLI**
    • dotnet restore
    • dotnet build --configuration Release`"]
    
    R --> T{"`**Build successful?**`"}
    S --> T
    T -->|No| U["`**BUILD FAILED - EXIT 1**`"]
    T -->|Yes| V["`**Copy outputs to BHoM folder**`"]
    
    V --> W{"`**More dependencies?**`"}
    W -->|Yes| O
    W -->|No| X["`**Upload Complete Dependencies**
    Replace incomplete artifact with complete set`"]
    
    X --> Y["`**Job Complete**
    dependencies-built=true`"]
    
    style A fill:#f3e5f5
    style F fill:#ffecb3
    style K fill:#ffecb3
    style Q fill:#ffecb3
    style T fill:#ffecb3
    style W fill:#ffecb3
    style U fill:#ffcdd2
```

---

## Tab 4: Dependency Artifact Download Process (Priority System)

```mermaid
graph TD
    A["`**Download Dependency Artifacts**
    Priority-based system`"] --> B["`**For each dependency:**`"]
    
    B --> C["`**Get Latest CI Build and Test Run**
    • Query GitHub API
    • Filter by branch and workflow name`"]
    
    C --> D{"`**Run found?**`"}
    D -->|No| E["`**Mark as missing**
    Add to missing-dependencies list`"]
    D -->|Yes| F["`**Check Run Status & Artifacts**
    Wait up to 30 attempts × 15 seconds`"]
    
    F --> G["`**Priority 1: Try Matching Configurations**
    For each release config:`"]
    G --> H{"`**Matching artifact found?**`"}
    H -->|Yes| I["`**Download Matching Artifact**
    build-outputs-$branch-$releaseConfig`"]
    H -->|No| J["`**Try next configuration**`"]
    
    J --> K{"`**More configs to try?**`"}
    K -->|Yes| G
    K -->|No| L["`**Priority 2: Try Release Fallback**
    Look for build-outputs-$branch-Release`"]
    
    L --> M{"`**Release artifact found?**`"}
    M -->|Yes| N["`**Download Release Artifact**
    Fallback to standard Release build`"]
    M -->|No| O["`**Mark as missing**
    No suitable artifacts available`"]
    
    I --> P["`**Extract & Copy**
    • Download ZIP
    • Extract to temp folder
    • Copy to BHoM folder (/tmp/BHoM)`"]
    N --> P
    
    P --> Q["`**Mark as found**
    Add to downloaded-artifacts list`"]
    
    E --> R{"`**More dependencies?**`"}
    O --> R
    Q --> R
    
    R -->|Yes| B
    R -->|No| S["`**Generate Summary**
    • Count downloaded vs missing
    • Set dependencies-built status`"]
    
    S --> T{"`**Any missing?**`"}
    T -->|Yes| U["`**Status: incomplete**
    Second job will run`"]
    T -->|No| V["`**Status: complete**
    Second job will be skipped`"]
    
    style D fill:#ffecb3
    style H fill:#ffecb3
    style K fill:#ffecb3
    style M fill:#ffecb3
    style T fill:#ffecb3
    style U fill:#ffcdd2
    style V fill:#c8e6c9
```

---

## Tab 5: build Job (Matrix) Detail

```mermaid
graph TD
    A["`**build Job (Matrix)**
    For each build-configuration
    Runs on: windows-latest`"] --> B["`**Setup Environment**
    • Checkout code
    • Setup MSBuild & NuGet
    • Create BHoM folder`"]
    
    B --> C{"`**Dependencies available?**`"}
    C -->|complete OR built| D["`**Download Dependencies artifact**
    Extract to BHoM folder`"]
    C -->|No| E["`**Skip dependency download**`"]
    
    D --> F["`**Copy libs folder**
    Copy DLL files to BHoM assemblies`"]
    E --> F
    
    F --> G["`**Take pre-build snapshot**
    Record existing files in BHoM folder`"]
    
    G --> H["`**Build Project**
    • Find .sln file
    • Detect project type
    • Try dotnet restore & build
    • Fallback to NuGet restore & MSBuild`"]
    
    H --> I{"`**Build successful?**`"}
    I -->|No| J["`**BUILD FAILED - EXIT 1**`"]
    I -->|Yes| K["`**Package Build Outputs**
    • Compare with pre-build snapshot
    • Identify changed/new files
    • Copy to artifact folder`"]
    
    K --> L["`**Upload Build Artifacts**
    Name: build-outputs-$branch-$config`"]
    
    L --> M{"`**Release configuration?**`"}
    M -->|Yes| N["`**Upload BHoM Assemblies**
    Name: bhom-assemblies-$config
    For test job consumption`"]
    M -->|No| O["`**Skip assemblies upload**`"]
    
    N --> P["`**Save Updated Assemblies List**
    Create updated-assemblies-$config.txt`"]
    O --> P
    
    P --> Q["`**Job Complete**`"]
    
    style A fill:#fff3e0
    style C fill:#ffecb3
    style I fill:#ffecb3
    style M fill:#ffecb3
    style J fill:#ffcdd2
```

---

## Tab 6: tests Job Detail

```mermaid
graph TD
    A["`**tests Job**
    Runs on: windows-latest
    Needs: setup-configurations-and-dependencies + build
    Condition: IF both jobs succeeded`"] --> B["`**Setup Environment**
    • Checkout code
    • Create BHoM directory`"]
    
    B --> C["`**Download and Merge BHoM Assemblies**
    Download all bhom-assemblies-* artifacts`"]
    
    C --> D["`**Download Test Toolkit Tests**
    Multi-branch priority system`"]
    
    D --> E["`**Convert Changed Files Paths**
    Convert relative paths to absolute Windows paths`"]
    
    E --> F["`**Identify Tests**
    • Find DLL files in _Tests_
    • Check for NUnit references
    • Pre-filter test assemblies`"]
    
    F --> G{"`**Test DLLs found?**`"}
    G -->|No| H["`**No tests to run - EXIT 1**`"]
    G -->|Yes| I["`**Run Individual Tests**
    For each test DLL:`"]
    
    I --> J{"`**Release builds available?**`"}
    J -->|Yes| K["`**Run tests per release build**
    With UpdatedAssemblies parameters`"]
    J -->|No| L["`**Run tests once**
    With UpdatedFiles parameters only`"]
    
    K --> M["`**Execute Tests (TRX format)**
    • dotnet test --logger trx
    • Generate test-results-$dll-$config.trx
    • Pass test parameters`"]
    L --> N["`**Execute Tests (TRX format)**
    • dotnet test --logger trx
    • Generate test-results-$dll.trx`"]
    
    M --> O{"`**All tests passed?**`"}
    N --> O
    O -->|No| P["`**Some tests failed - EXIT 1**`"]
    O -->|Yes| Q["`**Merge TRX files by test DLL**
    Group by DLL, merge configurations`"]
    
    Q --> R["`**Generate Dynamic Test Reports**
    Create composite action for dorny/test-reporter`"]
    
    R --> S["`**Execute Test Reports**
    • Use dorny/test-reporter@v2
    • Reporter: dotnet-trx
    • Generate GitHub check runs`"]
    
    S --> T["`**Job Complete**`"]
    
    style A fill:#fce4ec
    style G fill:#ffecb3
    style J fill:#ffecb3
    style O fill:#ffecb3
    style H fill:#ffcdd2
    style P fill:#ffcdd2
```

---

## Tab 7: Job Relationships and Flow

```mermaid
graph TD
    A["`**BHoM Build Workflow**
    Triggered by workflow_call
    Inputs: repository_name, branch_name, dotnet_version`"] --> B["`**Job 1: setup-configurations-and-dependencies**
    Runs on: ubuntu-latest
    Duration: ~2-3 minutes
    Purpose: Config discovery & dependency download`"]
    
    B --> C{"`**Dependencies complete?**`"}
    
    C -->|incomplete| D["`**Job 2: build-dependencies-with-no-artifacts**
    Runs on: windows-latest
    Duration: ~5-15 minutes (conditional)
    Purpose: Build missing dependencies only`"]
    
    C -->|complete| E["`**Skip Job 2**
    All dependencies already available`"]
    
    D --> F["`**Job 3: build (Matrix)**
    Runs on: windows-latest
    Matrix: build-configuration
    Duration: ~3-10 minutes per config
    Purpose: Build project`"]
    
    E --> F
    
    F --> G["`**Job 4: tests**
    Runs on: windows-latest
    Duration: ~2-5 minutes
    Purpose: Run tests & generate reports`"]
    
    B -.->|"build-configurations<br/>release-configurations<br/>dependencies-built<br/>changed-files<br/>missing-dependencies<br/>branch-map"| C
    
    B -.->|"Dependencies or<br/>Dependencies-Incomplete<br/>artifact"| D
    
    D -.->|"Dependencies artifact<br/>(complete set)<br/>dependencies-built=true"| F
    
    B -.->|"Configuration outputs<br/>changed-files"| G
    
    F -.->|"bhom-assemblies-*<br/>updated-assemblies-*.txt"| G
    
    H["`**Key Optimizations:**
    
    🚀 **Ubuntu First Job**: Faster, cheaper for config discovery
    ⚡ **Conditional Windows Job**: Only runs when needed
    🎯 **Priority Download System**: Matching configs → Release fallback
    🔄 **Cross-Platform Paths**: Ubuntu relative → Windows absolute
    📊 **Matrix Parallelization**: Multiple configurations simultaneously`"]
    
    style A fill:#e1f5fe
    style B fill:#e8f5e8
    style C fill:#ffecb3
    style D fill:#f3e5f5
    style E fill:#e0f2f1
    style F fill:#fff3e0
    style G fill:#fce4ec
    style H fill:#f1f8e9
```

---

## Tab 8: Cross-Platform Path Handling

```mermaid
graph TD
    A["`**Cross-Platform Path Challenge**
    Ubuntu job → Windows tests`"] --> B["`**Ubuntu Job (setup-configurations-and-dependencies)**
    Git diff generates relative paths`"]
    
    B --> C["`**Changed Files Processing**
    • git diff --name-only develop HEAD
    • Returns: TestA_Engine/Query/Method.cs
    • Store as relative paths (no $(pwd) prefix)`"]
    
    C --> D["`**Pass to Windows Job**
    changed-files=TestA_Engine/Query/Method.cs;Other/File.cs`"]
    
    D --> E["`**Windows Tests Job**
    Convert relative → absolute paths`"]
    
    E --> F["`**Path Conversion Logic**
    $workspaceRoot = Get-Location
    $absolutePath = Join-Path $workspaceRoot $relativePath`"]
    
    F --> G["`**Result**
    D:\a\TestRepo_A\TestRepo_A\TestA_Engine\Query\Method.cs`"]
    
    G --> H["`**Test Execution**
    Tests can now find files correctly`"]
    
    I["`**Why This Works:**
    
    ✅ **Ubuntu**: Generates portable relative paths
    ✅ **Windows**: Converts to proper Windows absolute paths
    ✅ **Cross-Platform**: No hardcoded platform-specific paths
    ✅ **Workspace Aware**: Uses current working directory`"]
    
    style A fill:#ffecb3
    style C fill:#e8f5e8
    style F fill:#fff3e0
    style G fill:#c8e6c9
    style I fill:#f1f8e9
```

---

## How to Use These Diagrams

1. **Copy each diagram section** from this file
2. **Create a new tab in Draw.io** for each diagram
3. **Import the Mermaid code** using Draw.io's Mermaid import feature
4. **Customize styling** as needed for your presentation

## Diagram Legend

- **🔷 Diamond shapes**: Decision points (if/else conditions)
- **📦 Rectangle shapes**: Process steps
- **🔴 Red styling**: Critical error conditions that exit the workflow
- **🟡 Yellow styling**: Decision points and conditional logic
- **🟢 Green styling**: Successful completion paths
- **⚡ Dotted arrows**: Data flow and dependencies between jobs
- **➡️ Solid arrows**: Process flow and execution order

## Key Features Illustrated

- **Split Job Architecture**: Ubuntu for configs, Windows for building when needed
- **Conditional Execution**: Second job only runs when dependencies are missing
- **Priority-Based Downloads**: Matching configurations → Release fallback
- **Cross-Platform Path Handling**: Ubuntu relative paths → Windows absolute paths
- **Matrix job execution** for parallel builds
- **Artifact dependencies** between jobs
- **Error handling** and exit conditions
- **Authentication hierarchy** (GitHub App → PAT → Default token)
- **Performance optimizations** (Ubuntu first, conditional building)
- **TRX test reporting** with dorny/test-reporter