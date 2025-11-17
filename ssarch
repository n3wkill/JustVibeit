#!/bin/bash

################################################################################
# ssarch
# Description: Fetches all Arch Linux mirrors and tests them for speed
# Author: Your Mind
# Disclaimer: This project is provided as-is with no warranty of any kind. I am not responsible for any damage, data loss, issues, or consequences that may result from using this software. Use it at your own risk.
# Requirements: curl, GNU parallel
################################################################################

# ============================================================================
# CONFIGURATION VARIABLES - Edit these to customize behavior
# ============================================================================

# URLs and endpoints
#readonly MIRROR_LIST_URL="https://archlinux.org/mirrorlist/all/"
#can, US, SWE
readonly MIRROR_LIST_URL="https://archlinux.org/mirrorlist/?country=CA&country=SE&country=US&protocol=https&ip_version=4&ip_version=6"
readonly TEST_FILE="core/os/x86_64/core.db"  # Small file to test download speed

# Testing parameters
readonly TOP_MIRRORS=3          # Number of fastest mirrors to return
readonly TIMEOUT=3               # Timeout in seconds for each mirror test
readonly MAX_PARALLEL=15        # Maximum parallel jobs for testing, 
readonly RETRY_COUNT=1           # Number of retries for failed connections

# Output file
readonly OUTPUT_FILE="/etc/pacman.d/mirrorlist"
readonly BACKUP_FILE="/etc/pacman.d/mirrorlist.backup"

# Catppuccin Mocha color scheme (dark theme)
readonly COLOR_BG="\e[48;2;30;30;46m"      # Base background
readonly COLOR_RESET="\e[0m"
readonly COLOR_TEXT="\e[38;2;205;214;244m"  # Text color
readonly COLOR_BLUE="\e[38;2;137;180;250m"  # Blue
readonly COLOR_GREEN="\e[38;2;166;227;161m" # Green
readonly COLOR_YELLOW="\e[38;2;249;226;175m" # Yellow
readonly COLOR_RED="\e[38;2;243;139;168m"   # Red
readonly COLOR_MAUVE="\e[38;2;203;166;247m" # Mauve
readonly COLOR_TEAL="\e[38;2;148;226;213m"  # Teal

# ============================================================================
# HELPER FUNCTIONS
# ============================================================================

# Function: print_header
# Description: Prints a styled header with title
# Parameters: $1 - Title text
print_header() {
    local title="$1"
    local width=80
    
    echo -e "${COLOR_MAUVE}"
    echo ""
    echo ""
    #    echo "═════════════════════════════════════════════════════════════════════"
    printf "%-*s\n" $width "  $title"
    echo ""

#    echo "═════════════════════════════════════════════════════════════════════"
    echo -e "${COLOR_RESET}"
}

# Function: print_info
# Description: Prints an info message
# Parameters: $1 - Message text
print_info() {
    echo -e "${COLOR_BLUE}[INFO]${COLOR_RESET} ${COLOR_TEXT}$1${COLOR_RESET}"
}

# Function: print_success
# Description: Prints a success message
# Parameters: $1 - Message text
print_success() {
    echo -e "${COLOR_GREEN}[SUCCESS]${COLOR_RESET} ${COLOR_TEXT}$1${COLOR_RESET}"
}

# Function: print_error
# Description: Prints an error message
# Parameters: $1 - Message text
print_error() {
    echo -e "${COLOR_RED}[ERROR]${COLOR_RESET} ${COLOR_TEXT}$1${COLOR_RESET}" >&2
}

# Function: print_warning
# Description: Prints a warning message
# Parameters: $1 - Message text
print_warning() {
    echo -e "${COLOR_YELLOW}[WARNING]${COLOR_RESET} ${COLOR_TEXT}$1${COLOR_RESET}"
}

# Function: check_root
# Description: Verifies script is running as root user
# Returns: Exits with error code 1 if not root
check_root() {
    if [[ $EUID -ne 0 ]]; then
        print_error "This script must be run as root (use sudo)"
        exit 1
    fi
}

# Function: check_dependencies
# Description: Checks if required commands are available
# Returns: Exits with error code 1 if dependencies missing
check_dependencies() {
    local missing_deps=()
    
    # Check for curl
    if ! command -v curl &> /dev/null; then
        missing_deps+=("curl")
    fi
    
    # Check for GNU parallel
    if ! command -v parallel &> /dev/null; then
        missing_deps+=("parallel")
    fi
    
    # If any dependencies are missing, report and exit
    if [[ ${#missing_deps[@]} -gt 0 ]]; then
        print_error "Missing required dependencies: ${missing_deps[*]}"
        print_info "Install with: pacman -S ${missing_deps[*]}"
        exit 1
    fi
}

# Function: parse_mirror_info
# Description: Extracts information about a mirror from its URL and metadata
# Parameters: $1 - Mirror URL
# Returns: Prints formatted string with mirror type information
parse_mirror_info() {
    local url="$1"
    local protocol=""
    local ip_version="IPv4"
    
    # Determine protocol (http or https)
    if [[ $url == https://* ]]; then
        protocol="HTTPS"
    elif [[ $url == http://* ]]; then
        protocol="HTTP"
    else
        protocol="Unknown"
    fi
    
    # Check if URL contains IPv6 address (enclosed in brackets)
    if [[ $url =~ \[.*:.*\] ]]; then
        ip_version="IPv6"
    fi
    
    echo "${protocol}/${ip_version}"
}

# Function: test_mirror_speed
# Description: Tests download speed of a single mirror
# Parameters: $1 - Mirror URL with country prefix (COUNTRY|||URL)
# Returns: Prints "URL|SPEED_IN_BYTES|COUNTRY|TYPE" or "URL|0|COUNTRY|TYPE" on failure
test_mirror_speed() {
    local input="$1"
    
    # Split input to get country and URL
    # Format: COUNTRY|||URL
    local country="${input%%%|||*}"
    local mirror_url="${input#*|||}"
    
    local test_url="${mirror_url}${TEST_FILE}"
    
    # Get mirror metadata
    local mirror_type=$(parse_mirror_info "$mirror_url")
    
    # Test download speed using curl
    # -o /dev/null: discard output
    # -s: silent mode
    # -S: show errors
    # -m: maximum time allowed
    # -w: write out format (speed_download gives bytes per second)
    local speed=$(curl -o /dev/null -s -S -m "$TIMEOUT" -w '%{speed_download}' "$test_url" 2>/dev/null)
    
    # Check if speed was successfully retrieved
    if [[ -z "$speed" ]] || [[ "$speed" == "0" ]] || [[ "$speed" == "0.000" ]]; then
        # Mirror failed or too slow
        echo "${mirror_url}|0|${country}|${mirror_type}"
    else
        # Convert speed to integer (remove decimal point)
        speed=${speed%.*}
        echo "${mirror_url}|${speed}|${country}|${mirror_type}"
    fi
}

# Function: format_speed
# Description: Converts bytes per second to human-readable format
# Parameters: $1 - Speed in bytes per second
# Returns: Formatted string (e.g., "1.5 MB/s")
format_speed() {
    local bytes=$1
    
    if [[ $bytes -eq 0 ]]; then
        echo "Failed"
        return
    fi
    
    # Convert to appropriate unit
    if [[ $bytes -lt 1024 ]]; then
        echo "${bytes} B/s"
    elif [[ $bytes -lt 1048576 ]]; then
        echo "$(awk "BEGIN {printf \"%.2f\", $bytes/1024}") KB/s"
    else
        echo "$(awk "BEGIN {printf \"%.2f\", $bytes/1048576}") MB/s"
    fi
}

# Function: export_function_for_parallel
# Description: Exports functions so GNU parallel can use them
export_function_for_parallel() {
    export -f test_mirror_speed
    export -f parse_mirror_info
    export TIMEOUT
    export TEST_FILE
}

# Function: confirm_write
# Description: Asks user for confirmation before writing mirrorlist
# Returns: 0 if user confirms, 1 if user cancels
confirm_write() {
    echo ""
#    echo -e "${COLOR_YELLOW}═════════════════════════════════════════════════════════════════════${COLOR_RESET}"
    echo -e "${COLOR_YELLOW}  CONFIRMATION REQUIRED${COLOR_RESET}"
#    echo -e "${COLOR_YELLOW}═════════════════════════════════════════════════════════════════════${COLOR_RESET}"
    echo ""
    echo -e "${COLOR_TEXT}This will:${COLOR_RESET}"
    echo -e "  ${COLOR_BLUE}1.${COLOR_RESET} Backup current mirrorlist to: ${COLOR_TEAL}${BACKUP_FILE}${COLOR_RESET}"
    echo -e "  ${COLOR_BLUE}2.${COLOR_RESET} Replace mirrorlist at: ${COLOR_TEAL}${OUTPUT_FILE}${COLOR_RESET}"
    echo -e "  ${COLOR_BLUE}3.${COLOR_RESET} Synchronization"
    echo ""
    echo -ne "${COLOR_TEXT}Do you want to proceed? ${COLOR_GREEN}[Y/n]${COLOR_RESET}: "
    
    read -r response
    
    # Default to Yes if empty response
    if [[ -z "$response" ]] || [[ "$response" =~ ^[Yy]$ ]]; then
        return 0
    else
        return 1
    fi
}

# ============================================================================
# MAIN SCRIPT LOGIC
# ============================================================================

# Clear screen and show header
clear
print_header "Arch Linux Mirror Speed Tester"

# Step 1: Check if running as root
print_info "Checking root privileges..."
check_root
print_success "Running as root"

# Step 2: Check dependencies
print_info "Checking dependencies..."
check_dependencies
print_success "All dependencies found"

# Step 3: Fetch mirror list
print_info "Fetching mirror list from $MIRROR_LIST_URL..."
mirror_page=$(curl -s "$MIRROR_LIST_URL")

if [[ -z "$mirror_page" ]]; then
    print_error "Failed to fetch mirror list"
    exit 1
fi

print_success "Mirror list fetched successfully"

# Step 4: Parse mirrors with their countries
print_info "Parsing mirror list..."

# Use associative array to map mirror URLs to countries
declare -A mirror_countries
current_country=""

# Read through the mirror list line by line
while IFS= read -r line; do
    # Check if line is a country marker (starts with ##)
    if [[ $line =~ ^##[[:space:]](.+)$ ]]; then
        # Extract country name
        current_country="${BASH_REMATCH[1]}"
        # Remove any extra spaces
        current_country=$(echo "$current_country" | xargs)
    # Check if line is a server definition
    elif [[ $line =~ ^#?Server[[:space:]]*=[[:space:]]*(.+)\$repo/os/\$arch$ ]]; then
        # Extract server URL
        server_url="${BASH_REMATCH[1]}"
        # Store the mapping
        if [[ -n "$current_country" ]]; then
            mirror_countries["$server_url"]="$current_country"
        else
            mirror_countries["$server_url"]="Unknown"
        fi
    fi
done <<< "$mirror_page"

# Create array of mirrors with country prefixes
declare -a mirrors_with_countries
for url in "${!mirror_countries[@]}"; do
    mirrors_with_countries+=("${mirror_countries[$url]}|||$url")
done

if [[ ${#mirrors_with_countries[@]} -eq 0 ]]; then
    print_error "No mirrors found in the list"
    exit 1
fi

print_success "Found ${#mirrors_with_countries[@]} mirrors"

# Step 5: Test mirror speeds using GNU parallel
print_info "Testing mirror speeds (this may take a while)..."
echo -e "${COLOR_TEAL}Progress will be shown below...${COLOR_RESET}\n"

# Export functions for parallel to use
export_function_for_parallel

# Create temporary file to store results
temp_results=$(mktemp)

# Use GNU parallel to test mirrors
# --bar: show progress bar
# --jobs: number of parallel jobs
# -k: keep order of results
printf '%s\n' "${mirrors_with_countries[@]}" | \
    parallel --bar \
             --jobs "$MAX_PARALLEL" \
             --progress \
             --eta \
             test_mirror_speed {} > "$temp_results"

# Step 6: Parse and sort results
print_info "Analyzing results..."

# Read results and sort by speed (descending)
# Format: URL|SPEED|COUNTRY|TYPE
declare -a sorted_results

while IFS='|' read -r url speed country type; do
    sorted_results+=("$speed|$url|$country|$type")
done < "$temp_results"

# Sort by speed (first field, numeric, reverse order)
IFS=$'\n' sorted_results=($(sort -t'|' -k1 -nr <<<"${sorted_results[*]}"))
unset IFS

# Step 7: Display top mirrors
echo ""
print_header "Top $TOP_MIRRORS Fastest Mirrors"
echo ""

declare -a top_mirrors

for i in $(seq 0 $((TOP_MIRRORS - 1))); do
    if [[ $i -ge ${#sorted_results[@]} ]]; then
        break
    fi
    
    IFS='|' read -r speed url country type <<< "${sorted_results[$i]}"
    
    if [[ "$speed" -eq 0 ]]; then
        print_warning "Not enough working mirrors found"
        break
    fi
    
    top_mirrors+=("$url")
    
    formatted_speed=$(format_speed "$speed")
    
    echo -e "${COLOR_GREEN}[$(($i + 1))]${COLOR_RESET} ${COLOR_BLUE}$url${COLOR_RESET}"
    echo -e "    ${COLOR_YELLOW}Speed:${COLOR_RESET}    ${formatted_speed}"
    echo -e "    ${COLOR_YELLOW}Location:${COLOR_RESET} ${country}"
    echo -e "    ${COLOR_YELLOW}Type:${COLOR_RESET}     ${type}"
    echo ""
done

# Step 8: Ask for confirmation before writing
if ! confirm_write; then
    print_warning "Operation cancelled by user"
    print_info "No changes were made to your mirrorlist"
    rm -f "$temp_results"
    exit 0
fi

# Step 9: Backup existing mirrorlist
if [[ -f "$OUTPUT_FILE" ]]; then
    print_info "Backing up existing mirrorlist..."
    cp "$OUTPUT_FILE" "$BACKUP_FILE"
    print_success "Backup created at $BACKUP_FILE"
fi

# Step 10: Write to mirrorlist file
print_info "Writing to $OUTPUT_FILE..."

{
    echo "# Arch Linux mirrorlist"
    echo "# Generated by Arch Mirror Speed Tester on $(date)"
    echo "# Top $TOP_MIRRORS fastest mirrors"
    echo ""
    
    for i in "${!top_mirrors[@]}"; do
        IFS='|' read -r speed url country type <<< "${sorted_results[$i]}"
        formatted_speed=$(format_speed "$speed")
        
        echo "# Mirror $((i + 1)): $country - $type - $formatted_speed"
        echo "Server = ${url}\$repo/os/\$arch"
        echo ""
    done
} > "$OUTPUT_FILE"
pacman -Syy
print_success "Mirrorlist updated successfully!"

# Step 11: Cleanup
rm -f "$temp_results"

# Final message
echo ""
print_header "Complete!"
#echo -e "${COLOR_TEXT}Your mirrorlist has been updated with the $TOP_MIRRORS fastest mirrors.${COLOR_RESET}"
#echo -e "${COLOR_TEXT}You can now run: ${COLOR_GREEN}pacman -Syy${COLOR_TEXT} to sync with new mirrors${COLOR_RESET}"
echo ""

