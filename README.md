## Step 1: Generate a Personal Access Token in Bitbucket
1. Log in to your Bitbucket account
2. Go to your account settings (click on your profile picture → Settings)
3. In the left sidebar, click on "Access tokens" under "Access Management"
4. Click "Create token"
5. Give your token a name (e.g., "Automation Script")
6. Select the appropriate permissions:
   - At minimum, you need "Repository read" permission
   - If your script needs to push changes, also add "Repository write"
7. Set an expiration date (or leave it as "Never expires" for automation purposes)
8. Click "Create"
9. Important : Copy the token immediately as you won't be able to see it again


## Step 2: Update Your Configuration File
 test_config.conf file to use token authentication:

***********************************************
test_config.conf
***********************************************
# Repository configuration
REPO_URL=git@bitbucket.org:username/project.git
BRANCH_NAME=main

# SSH Authentication
SSH_KEY_PATH=/path/to/your/private/key

# Java and Maven configuration
JAVA_HOME=/path/to/your/specific/jdk
MAVEN_HOME=/path/to/your/specific/maven

# TestNG XML files (one per line)
TESTNG_FILES=src/test/resources/testng1.xml
TESTNG_FILES=src/test/resources/testng2.xml

*********************************************


Important notes:

- Use the HTTPS URL format for the repository
- Enter your Bitbucket username in AUTH_USERNAME
- Enter your personal access token in AUTH_PASSWORD

  ## Step 3: Ensure Proper File Permissions
Since your configuration file now contains a sensitive token, make sure it has restricted permissions:

chmod 600 test_config.conf

This ensures only the file owner can read or write to the file.

-------------------------
Create run_tests.sh 
-------------------------

#!/bin/bash

# Configuration variables
CONFIG_FILE="test_config.conf"
LOG_DIR="./logs"
BUILD_LOG="${LOG_DIR}/build.log"
TEST_LOG="${LOG_DIR}/test.log"
SUMMARY_LOG="${LOG_DIR}/summary.log"

# Function to log messages to console and log file
log() {
    local message="$1"
    local log_file="$2"
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $message" | tee -a "$log_file"
}

# Check if config file exists
if [ ! -f "$CONFIG_FILE" ]; then
    echo "Error: Configuration file '$CONFIG_FILE' not found."
    echo "Please create a configuration file with the required parameters."
    exit 1
fi

# Create log directory if it doesn't exist
mkdir -p "$LOG_DIR"

# Initialize log files
> "$BUILD_LOG"
> "$TEST_LOG"
> "$SUMMARY_LOG"

# Read configuration from file
REPO_URL=""
BRANCH_NAME=""
SSH_KEY_PATH=""
JAVA_HOME=""
MAVEN_HOME=""
TESTNG_FILES=()

while IFS='=' read -r key value || [ -n "$key" ]; do
    # Skip comments and empty lines
    [[ $key =~ ^[[:space:]]*# ]] && continue
    [[ -z "$key" ]] && continue
    
    # Remove leading/trailing whitespace
    key=$(echo "$key" | xargs)
    value=$(echo "$value" | xargs)
    
    case "$key" in
        REPO_URL)
            REPO_URL="$value"
            ;;
        BRANCH_NAME)
            BRANCH_NAME="$value"
            ;;
        SSH_KEY_PATH)
            SSH_KEY_PATH="$value"
            ;;
        JAVA_HOME)
            JAVA_HOME="$value"
            ;;
        MAVEN_HOME)
            MAVEN_HOME="$value"
            ;;
        TESTNG_FILES)
            TESTNG_FILES+=("$value")
            ;;
    esac
done < "$CONFIG_FILE"

# Validate required configuration
if [ -z "$REPO_URL" ]; then
    log "ERROR: Repository URL not specified in configuration file." "$SUMMARY_LOG"
    exit 1
fi

if [ -z "$BRANCH_NAME" ]; then
    log "ERROR: Branch name not specified in configuration file." "$SUMMARY_LOG"
    exit 1
fi

if [ -z "$SSH_KEY_PATH" ]; then
    log "ERROR: SSH key path not specified in configuration file." "$SUMMARY_LOG"
    exit 1
fi

if [ ! -f "$SSH_KEY_PATH" ]; then
    log "ERROR: SSH key not found at $SSH_KEY_PATH" "$SUMMARY_LOG"
    exit 1
fi

if [ ${#TESTNG_FILES[@]} -eq 0 ]; then
    log "ERROR: No TestNG XML files specified in configuration file." "$SUMMARY_LOG"
    exit 1
fi

# Validate and set up Java and Maven environment
if [ -n "$JAVA_HOME" ]; then
    if [ ! -d "$JAVA_HOME" ]; then
        log "WARNING: Specified JAVA_HOME directory does not exist: $JAVA_HOME" "$SUMMARY_LOG"
    else
        log "Using Java from: $JAVA_HOME" "$SUMMARY_LOG"
        export JAVA_HOME
        export PATH="$JAVA_HOME/bin:$PATH"
    fi
fi

if [ -n "$MAVEN_HOME" ]; then
    if [ ! -d "$MAVEN_HOME" ]; then
        log "WARNING: Specified MAVEN_HOME directory does not exist: $MAVEN_HOME" "$SUMMARY_LOG"
    else
        log "Using Maven from: $MAVEN_HOME" "$SUMMARY_LOG"
        export MAVEN_HOME
        export PATH="$MAVEN_HOME/bin:$PATH"
    fi
fi

# Verify Java and Maven versions
log "Java version:" "$SUMMARY_LOG"
java -version 2>&1 | tee -a "$SUMMARY_LOG"

log "Maven version:" "$SUMMARY_LOG"
mvn --version 2>&1 | tee -a "$SUMMARY_LOG"

# Extract repository name from URL
REPO_NAME=$(basename "$REPO_URL" .git)

log "Starting process for repository: $REPO_NAME" "$SUMMARY_LOG"
log "Using branch: $BRANCH_NAME" "$SUMMARY_LOG"
log "TestNG files to run: ${TESTNG_FILES[*]}" "$SUMMARY_LOG"

# Configure SSH for git operations
export GIT_SSH_COMMAND="ssh -i $SSH_KEY_PATH -o StrictHostKeyChecking=no"

# Check if repository already exists
if [ -d "$REPO_NAME" ]; then
    log "Repository already exists, updating from remote..." "$SUMMARY_LOG"
    cd "$REPO_NAME" || {
        log "ERROR: Failed to change to repository directory." "$SUMMARY_LOG"
        exit 1
    }
    
    # Fetch and reset to the specified branch
    git fetch origin 2>&1 | tee -a "../$BUILD_LOG"
    git reset --hard "origin/$BRANCH_NAME" 2>&1 | tee -a "../$BUILD_LOG"
    UPDATE_EXIT_CODE=${PIPESTATUS[1]}
    
    if [ $UPDATE_EXIT_CODE -ne 0 ]; then
        log "ERROR: Failed to update repository. Check $BUILD_LOG for details." "$SUMMARY_LOG"
        exit 1
    }
else
    # Clone the repository
    log "Cloning repository from $REPO_URL (branch: $BRANCH_NAME)..." "$SUMMARY_LOG"
    git clone -b "$BRANCH_NAME" "$REPO_URL" 2>&1 | tee -a "$BUILD_LOG"
    CLONE_EXIT_CODE=${PIPESTATUS[0]}
    
    if [ $CLONE_EXIT_CODE -ne 0 ]; then
        log "ERROR: Failed to clone repository. Check $BUILD_LOG for details." "$SUMMARY_LOG"
        exit 1
    fi
    
    cd "$REPO_NAME" || {
        log "ERROR: Failed to change to repository directory." "$SUMMARY_LOG"
        exit 1
    }
fi

# Check for project-specific Maven settings
if [ -f "mvnw" ]; then
    log "Using project-specific Maven wrapper" "$SUMMARY_LOG"
    MVN_CMD="./mvnw"
    chmod +x mvnw
else
    MVN_CMD="mvn"
fi

# Clean and compile the project
log "Cleaning and compiling the project..." "$SUMMARY_LOG"
$MVN_CMD clean compile 2>&1 | tee -a "../$BUILD_LOG"

if [ $? -ne 0 ]; then
    log "ERROR: Build failed. Check $BUILD_LOG for details." "$SUMMARY_LOG"
    exit 1
fi

log "Build successful." "$SUMMARY_LOG"

# Run tests for each TestNG XML file
TOTAL_TESTS=0
PASSED_TESTS=0
FAILED_TESTS=0

for testng_xml in "${TESTNG_FILES[@]}"; do
    log "Running tests from $testng_xml..." "$SUMMARY_LOG"
    
    # Create a separate log file for each test suite
    TEST_SUITE_NAME=$(basename "$testng_xml" .xml)
    TEST_SUITE_LOG="../${LOG_DIR}/${TEST_SUITE_NAME}_results.log"
    
    # Run the tests
    $MVN_CMD test -DsuiteXmlFile="$testng_xml" 2>&1 | tee "$TEST_SUITE_LOG"
    
    TEST_EXIT_CODE=${PIPESTATUS[0]}
    
    # Extract test results
    TESTS_RUN=$(grep -o "Tests run: [0-9]*" "$TEST_SUITE_LOG" | awk '{sum+=$3} END {print sum}')
    TESTS_FAILED=$(grep -o "Failures: [0-9]*" "$TEST_SUITE_LOG" | awk '{sum+=$2} END {print sum}')
    TESTS_SKIPPED=$(grep -o "Skipped: [0-9]*" "$TEST_SUITE_LOG" | awk '{sum+=$2} END {print sum}')
    
    if [ -z "$TESTS_RUN" ]; then TESTS_RUN=0; fi
    if [ -z "$TESTS_FAILED" ]; then TESTS_FAILED=0; fi
    if [ -z "$TESTS_SKIPPED" ]; then TESTS_SKIPPED=0; fi
    
    TESTS_PASSED=$((TESTS_RUN - TESTS_FAILED - TESTS_SKIPPED))
    
    # Update totals
    TOTAL_TESTS=$((TOTAL_TESTS + TESTS_RUN))
    PASSED_TESTS=$((PASSED_TESTS + TESTS_PASSED))
    FAILED_TESTS=$((FAILED_TESTS + TESTS_FAILED))
    
    # Log test results
    log "Test suite: $TEST_SUITE_NAME" "$SUMMARY_LOG"
    log "  Tests run: $TESTS_RUN, Passed: $TESTS_PASSED, Failed: $TESTS_FAILED, Skipped: $TESTS_SKIPPED" "$SUMMARY_LOG"
    
    if [ $TEST_EXIT_CODE -ne 0 ]; then
        log "  Status: FAILED" "$SUMMARY_LOG"
    else
        log "  Status: PASSED" "$SUMMARY_LOG"
    fi
    
    # Append to main test log
    cat "$TEST_SUITE_LOG" >> "../$TEST_LOG"
done

# Generate summary
log "========== TEST EXECUTION SUMMARY ==========" "$SUMMARY_LOG"
log "Total tests run: $TOTAL_TESTS" "$SUMMARY_LOG"
log "Tests passed: $PASSED_TESTS" "$SUMMARY_LOG"
log "Tests failed: $FAILED_TESTS" "$SUMMARY_LOG"

if [ $FAILED_TESTS -gt 0 ]; then
    log "OVERALL STATUS: FAILED" "$SUMMARY_LOG"
    EXIT_CODE=1
else
    log "OVERALL STATUS: PASSED" "$SUMMARY_LOG"
    EXIT_CODE=0
fi

log "Detailed logs:" "$SUMMARY_LOG"
log "  Build log: $BUILD_LOG" "$SUMMARY_LOG"
log "  Test log: $TEST_LOG" "$SUMMARY_LOG"
log "  Individual test suite logs are in the $LOG_DIR directory" "$SUMMARY_LOG"

# Return to original directory
cd ..

exit $EXIT_CODE



## Step 5: Script Make sure to set the script as executable with:

chmod +x run_tests.sh



## Step 6: Run the Script
Now you can run the script as before:

./run_tests.sh


---------------------------------------
# RunningTestFromUnix
Running Test using MVN in Unix Host


## Features of the Script
1. Comprehensive Logging :
   
   - Build logs in logs/build.log
   - Test execution logs in logs/test.log
   - Individual test suite logs in logs/[test_suite_name]_results.log
   - Summary log with overall results in logs/summary.log
2. Error Handling :
   
   - Checks for required arguments
   - Validates each step (clone, build, test)
   - Reports failures with specific error messages
3. Test Results Summary :
   
   - Counts total tests, passed tests, and failed tests
   - Provides a clear summary at the end of execution
4. Extensibility :
   
   - Can run multiple TestNG XML files in sequence
   - Easy to add more test files as needed
5. Debugging Support :
   
   - Detailed logs for each phase
   - Separate logs for each test suite for easier debugging
You can further customize this script based on your specific requirements, such as adding email notifications for test results or integrating with a CI/CD system.
