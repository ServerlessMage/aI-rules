# Shell Script Color and Formatting Guidelines

When writing shell scripts, use color and formatting to improve readability and user experience.

## Color Definitions

Always define these standard colors at the beginning of your script:

```bash
# Define colors for better readability
GREEN='\033[0;32m'
BLUE='\033[0;34m'
YELLOW='\033[1;33m'
RED='\033[0;31m'
CYAN='\033[0;36m'
BOLD='\033[1m'
NC='\033[0m' # No Color (reset)
```

## Color Usage Guidelines

Use specific colors for specific types of messages:

- **GREEN**: Success messages, completed operations
  ```bash
  echo -e "${GREEN}✓ Operation completed successfully${NC}"
  ```

- **BLUE**: Informational messages, descriptions
  ```bash
  echo -e "${BLUE}Processing file: ${YELLOW}$filename${NC}"
  ```

- **YELLOW**: Highlight important values, paths, filenames
  ```bash
  echo -e "Loading configuration from ${YELLOW}$config_path${NC}"
  ```

- **RED**: Error messages, warnings
  ```bash
  echo -e "${RED}Error: Failed to connect to server${NC}"
  ```

- **CYAN**: Section headers, major steps
  ```bash
  echo -e "${CYAN}=== INITIALIZING SYSTEM ===${NC}"
  ```

## Structured Output

For complex scripts, organize output into clear sections:

```bash
# Function to print section headers
print_header() {
    echo -e "\n${BOLD}${CYAN}=== $1 ===${NC}\n"
}

# Usage
print_header "SETUP"
print_header "PROCESSING"
print_header "SUMMARY"
```

## List Formatting

For summary information, use bullet points with consistent indentation:

```bash
echo -e "${GREEN}✓ Process complete.${NC}"
echo -e "  ${BLUE}• Files processed: ${YELLOW}$file_count${NC}"
echo -e "  ${BLUE}• Errors encountered: ${YELLOW}$error_count${NC}"
```

## Important Notes

1. Always reset color with `${NC}` at the end of colored text
2. Use `-e` flag with echo to interpret escape sequences
3. Test your scripts in different terminal environments to ensure readability
4. Consider users with color blindness by using distinct formatting (bold, indentation) in addition to colors
5. For scripts that may be run in non-interactive environments, provide a `--no-color` option to disable colors