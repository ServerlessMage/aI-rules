# Shell Scripting Best Practices

## not using localstack

if the script is using aws cli and is NOT for local stack add

```
export AWS_PROFILE="${AWS_PROFILE}"
export AWS_REGION="${AWS_REGION}"

echo "=== Using AWS Profile: ${AWS_PROFILE} in Region: ${AWS_REGION} ==="
```

to the script and do not use `--endpoint-url parameter` on CLI commands for AWS otherwise the endpoint
can not be found as it will try and use localstack

## Always Quote Variables

Always enclose variables in double quotes when using them in shell scripts:

```bash
# BAD
file=$1
cat $file

# GOOD
file="$1"
cat "$file"
```

By enclosing the variable in double quotes, you ensure that its entire value is passed as a single argument to commands, regardless of any spaces or special characters it might contain.

This prevents:

- Word splitting (when variables contain spaces)
- Globbing (when variables contain wildcards like \* or ?)
- Command injection vulnerabilities

## Examples

```bash
# BAD - will break if filename contains spaces
filename=my file.txt
rm $filename

# GOOD - works with any filename
filename="my file.txt"
rm "$filename"
```

```bash
# BAD - command substitution without quotes
output=$(find . -name "*.log")
for file in $output; do
  echo "Processing $file"
done

# GOOD - preserves spaces and special characters
output=$(find . -name "*.log")
for file in "$output"; do
  echo "Processing $file"
done
```

Always quote variables in:

- Command arguments: `grep "pattern" "$file"`
- Conditionals: `if [ "$count" -eq 0 ]`
- Assignments: `result="$value"`
- Loops: `for item in "$array"`

The only exceptions are when you specifically need word splitting, such as in array construction or when using a variable as multiple arguments.