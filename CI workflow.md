# BHoM CI Workflow Diagrams

This file contains all the diagrams explaining the BHoM Build Workflow. Each section represents a separate tab/diagram that can be imported into Draw.io or other diagramming tools.

---

## Tab 1: Main Overview - BHoM Build Workflow

```mermaid
graph TD
    A["`**🚀 BHoM Build Workflow**
    
    **Triggers:** workflow_call
    **Inputs:** repository_name, branch_name, dotnet_version
    **Concurrency:** Cancel in-progress runs on new commits`"] --> B["`**📦 Job 1: prepare-dependencies**
    **Purpose:** Setup all dependencies and test files
    **Duration:** ~2-5 minutes
    **Runs on:** windows-latest
    
    📋 **See Detailed Diagram 2**`"]
    
    B --> C["`**🔨 Job 2: build (Matrix)**
    **Purpose:** Build project for each configuration
    **Matrix:** build-configuration (Debug, Release, etc.)
    **Duration:** ~3-10 minutes per config
    **Runs on:** windows-latest
    
    📋 **See Detailed Diagram 5**`"]
    
    C --> D["`**📊 Job 3: test-report (Matrix)**
    **Purpose:** Generate test reports for GitHub
    **Matrix:** test-dll (each test library)
    **Duration:** ~30 seconds per DLL
    **Runs on:** ubuntu-latest
    
    📋 **See Detailed Diagram 6**`"]
    
    B -.->|"`**Artifacts & Outputs:**
    • build-configurations
    • release-configurations  
    • dependencies-built
    • changed-files
    • Dependencies (artifact)
    • Test-Toolkit-Tests (artifact)`"| C
    
    C -.->|"`**Artifacts & Outputs:**
    • test-dll-names
    • build-outputs-$branch-$config
    • test-results-$config`"| D
    
    E["`**🔍 Key Sub-processes:**
    
    **Diagram 3:** Dependency Artifact Download
    **Diagram 4:** Test Toolkit Tests Download
    
    **⚠️ Critical Error Conditions:**
    • Missing expected dependency artifacts → EXIT 1
    • Build failures → EXIT 1  
    • No test DLLs found → EXIT 1
    • Test failures → EXIT 1`"]
    
    F["`**📈 Performance Optimizations:**
    
    • **Parallel artifact checking** while workflows run
    • **Matrix builds** for multiple configurations
    • **Artifact caching** to avoid rebuilding dependencies
    • **Early termination** on critical errors
    • **Concurrency control** to cancel outdated runs`"]
    
    G["`**🔧 Authentication & Access:**
    
    • **GitHub App authentication** (preferred)
    • **Personal Access Token** fallback
    • **Default GITHUB_TOKEN** as last resort
    • **Cross-repository access** for dependencies`"]
    
    style A fill:#e3f2fd
    style B fill:#f3e5f5
    style C fill:#e8f5e8
    style D fill:#fff3e0
    style E fill:#fce4ec
    style F fill:#e0f2f1
    style G fill:#f1f8e9
```

---

## Tab 2: prepare-dependencies Job Detail

```mermaid
graph TD
    A["`**prepare-dependencies Job**`"] --> B["`**Discover Build Configurations**
    • Parse .sln file
    • Extract build configs
    • Identify Release configs`"]
    
    B --> C["`**Parse Dependencies**
    • Read dependencies.txt
    • Extract dependency list`"]
    
    C --> D["`**Get Changed Files**
    Compare with develop branch`"]
    
    D --> E{"`**Branch = develop?**`"}
    E -->|Yes| F["`**No changed files tracking**`"]
    E -->|No| G["`**Git diff vs develop**
    • Fetch develop branch
    • Compare files
    • Generate file list`"]
    
    F --> H["`**Setup Authentication**
    • Check GitHub App secrets
    • Generate token if available`"]
    G --> H
    
    H --> I["`**Determine Dependency Branches**
    For each dependency:`"]
    I --> J{"`**Target branch exists?**`"}
    J -->|Yes| K["`**Use target branch**`"]
    J -->|No| L["`**Use develop branch**`"]
    
    K --> M["`**Download Dependency Artifacts**
    See detailed diagram`"]
    L --> M
    
    M --> N{"`**Artifacts found?**`"}
    N -->|Yes| O["`**Extract to BHoM folder**`"]
    N -->|No| P["`**Clone Missing Dependencies**
    See detailed diagram`"]
    
    O --> Q["`**Download Test Toolkit Tests**
    See detailed diagram`"]
    P --> R["`**Build Missing Dependencies**
    • Analyze dependency order
    • Topological sort
    • Build in correct order`"]
    
    R --> Q
    Q --> S["`**Upload Artifacts**
    • Dependencies (if built)
    • Test-Toolkit-Tests`"]
    
    style A fill:#f3e5f5
    style E fill:#ffecb3
    style J fill:#ffecb3
    style N fill:#ffecb3
```

---

## Tab 3: Dependency Artifact Download Process

```mermaid
graph TD
    A["`**Download Dependency Artifacts**
    For each dependency`"] --> B["`**Get Latest CI Build and Test Run**
    • Query GitHub API
    • Filter by branch and workflow name`"]
    
    B --> C{"`**Run found?**`"}
    C -->|No| D["`**Mark for building**`"]
    C -->|Yes| E["`**Check Run Status**
    Status: $status, Conclusion: $conclusion`"]
    
    E --> F{"`**Run completed?**`"}
    F -->|Yes| G["`**Check for artifacts immediately**`"]
    F -->|No| H["`**Start wait loop**
    Max 10 attempts, 10s each`"]
    
    H --> I["`**Wait 10 seconds**`"]
    I --> J["`**Check run status**`"]
    J --> K["`**Check for artifacts**`"]
    K --> L{"`**Required artifacts found?**`"}
    
    L -->|Yes| M["`**Download & Extract**
    • Download artifact ZIP
    • Extract to dependencies folder
    • Copy to BHoM folder`"]
    
    L -->|No| N{"`**Run completed?**`"}
    N -->|Yes| O{"`**Has any artifacts?**`"}
    N -->|No| P{"`**Max attempts reached?**`"}
    
    O -->|Yes| Q["`**CRITICAL ERROR**
    Expected artifacts not found
    List available artifacts
    EXIT 1`"]
    O -->|No| R["`**CRITICAL ERROR**
    No artifacts in completed run
    EXIT 1`"]
    
    P -->|Yes| S["`**Mark for building**
    Timeout reached`"]
    P -->|No| H
    
    G --> L
    M --> T["`**Success - Continue to next dependency**`"]
    D --> U["`**Continue to clone & build phase**`"]
    S --> U
    
    style C fill:#ffecb3
    style F fill:#ffecb3
    style L fill:#ffecb3
    style N fill:#ffecb3
    style O fill:#ffecb3
    style P fill:#ffecb3
    style Q fill:#ffcdd2
    style R fill:#ffcdd2
```

---

## Tab 4: Test Toolkit Tests Download

```mermaid
graph TD
    A["`**Download Test Toolkit Tests**
    Repository: BHoM/Test_Toolkit`"] --> B["`**Branch Priority Order**
    1. ${{ inputs.branch_name }}
    2. develop
    3. Test_Toolkit-InvestigateRunningAllChecksThroughNUnit`"]
    
    B --> C["`**For each branch in priority order:**`"]
    C --> D["`**Get successful runs from branch**`"]
    D --> E{"`**Runs found?**`"}
    
    E -->|No| F["`**Try next branch**`"]
    E -->|Yes| G["`**Check latest run for Tests artifact**`"]
    
    G --> H{"`**Tests artifact found?**`"}
    H -->|Yes| I["`**Download & Extract**
    • Download Tests.zip
    • Extract to _Tests_ folder
    • Clean up ZIP file`"]
    H -->|No| F
    
    I --> J["`**Success - Upload as Test-Toolkit-Tests artifact**`"]
    
    F --> K{"`**More branches to try?**`"}
    K -->|Yes| C
    K -->|No| L["`**Create empty _Tests_ folder**
    Workflow can continue without tests`"]
    
    L --> M["`**Upload empty Test-Toolkit-Tests artifact**`"]
    J --> N["`**Continue to build job**`"]
    M --> N
    
    style E fill:#ffecb3
    style H fill:#ffecb3
    style K fill:#ffecb3
    style L fill:#fff3e0
```

---

## Tab 5: build Job (Matrix) Detail

```mermaid
graph TD
    A["`**build Job (Matrix)**
    For each build-configuration`"] --> B["`**Setup Environment**
    • Checkout code
    • Setup MSBuild & NuGet
    • Create BHoM folder`"]
    
    B --> C{"`**Dependencies built?**`"}
    C -->|Yes| D["`**Download Dependencies artifact**
    Extract to BHoM folder`"]
    C -->|No| E["`**Skip dependency download**`"]
    
    D --> F["`**Copy libs folder**
    Copy DLL files to BHoM assemblies`"]
    E --> F
    
    F --> G["`**Take pre-build snapshot**
    Record existing files in BHoM folder`"]
    
    G --> H["`**Build Project**
    • Find .sln file
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
    
    L --> M["`**Download Test Toolkit Tests**
    Download Test-Toolkit-Tests artifact`"]
    
    M --> N["`**Identify Tests**
    • Find DLL files in _Tests_
    • Check for NUnit references
    • Verify actual test methods exist`"]
    
    N --> O{"`**Test DLLs found?**`"}
    O -->|No| P["`**No tests to run - EXIT 1**`"]
    O -->|Yes| Q["`**Run Individual Tests**
    For each test DLL:`"]
    
    Q --> R["`**Execute Tests**
    • Add test parameters if available
    • Generate TRX log files
    • Track pass/fail status`"]
    
    R --> S{"`**All tests passed?**`"}
    S -->|No| T["`**Some tests failed - EXIT 1**`"]
    S -->|Yes| U["`**Upload Test Results**
    Upload TRX files as artifacts`"]
    
    U --> V["`**Job Complete**`"]
    
    style C fill:#ffecb3
    style I fill:#ffecb3
    style O fill:#ffecb3
    style S fill:#ffecb3
    style J fill:#ffcdd2
    style P fill:#ffcdd2
    style T fill:#ffcdd2
```

---

## Tab 6: test-report Job (Matrix) Detail

```mermaid
graph TD
    A["`**test-report Job (Matrix)**
    For each test-dll from build job`"] --> B{"`**Test DLLs exist?**`"}
    
    B -->|No| C["`**Skip - No tests to report**`"]
    B -->|Yes| D["`**Setup Environment**
    • Checkout repository
    • Run on ubuntu-latest`"]
    
    D --> E["`**Download Test Results**
    • Download all test-results-* artifacts
    • Merge multiple artifacts`"]
    
    E --> F["`**Create Test Report**
    • Use dorny/test-reporter@v1
    • Generate report for specific test DLL
    • Format: dotnet-trx
    • List failed tests only`"]
    
    F --> G["`**Publish Report**
    • Create GitHub check
    • Show test results in PR/commit
    • Include pass/fail summary`"]
    
    G --> H["`**Job Complete**`"]
    C --> H
    
    style B fill:#ffecb3
```

---

## Tab 7: Job Relationships and Flow

```mermaid
graph TD
    A["`**BHoM Build Workflow**
    Triggered by workflow_call
    Inputs: repository_name, branch_name, dotnet_version`"] --> B["`**Job 1: prepare-dependencies**
    Runs on: windows-latest
    Purpose: Setup dependencies and test artifacts`"]
    
    B --> C["`**Job 2: build**
    Runs on: windows-latest
    Matrix: build-configuration
    Purpose: Build and test the project`"]
    
    C --> D["`**Job 3: test-report**
    Runs on: ubuntu-latest
    Matrix: test-dll
    Purpose: Generate test reports`"]
    
    B -.->|"Outputs: build-configurations<br/>release-configurations<br/>dependencies-built<br/>changed-files"| C
    C -.->|"Outputs: test-dll-names"| D
    
    style A fill:#e1f5fe
    style B fill:#f3e5f5
    style C fill:#e8f5e8
    style D fill:#fff3e0
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

- **Concurrency control** and workflow cancellation
- **Matrix job execution** for parallel builds
- **Artifact dependencies** between jobs
- **Error handling** and exit conditions
- **Authentication hierarchy** (GitHub App → PAT → Default token)
- **Performance optimizations** (parallel checking, early termination)
- **Branch prioritization** for test toolkit downloads
- **Wait loops** for running workflows
