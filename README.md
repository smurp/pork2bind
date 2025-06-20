# pork2bind

Convert Porkbun DNS records to BIND zone files with a simple two-phase workflow.

## Features

- **Two-phase operation**: Fetch DNS data from Porkbun API, then convert to BIND format
- **Batch processing**: Handle multiple domains at once
- **Smart filtering**: Automatically filters out external NS records (with override option)
- **Flexible directories**: Control input/output locations
- **Standard BIND format**: Generates properly formatted zone files with timestamp-based serials
- **Clean output control**: Quiet mode and stdout output for scripting and automation

## Installation

1. Clone this repository:
```bash
git clone https://github.com/smurp/pork2bind.git
cd pork2bind
```

2. Install dependencies:
```bash
npm install
```

3. Make executable:
```bash
chmod +x pork2bind
```

4. Create your `.env` file:
```bash
cp .env.example .env
# Edit .env with your Porkbun API credentials
```

## Configuration

Create a `.env` file in the project directory:

```bash
PORKBUN_API_KEY=pk1_your_api_key_here
PORKBUN_SECRET_KEY=sk1_your_secret_key_here
```

### Getting Porkbun API Keys

1. Log into your [Porkbun account](https://porkbun.com)
2. Go to Account Settings → API Access
3. Generate your API key and secret key
4. Enable API access for your domains

## Usage

### Phase 1: Fetch DNS Records

Retrieve DNS records from Porkbun and save as JSON files:

```bash
# Single domain
pork2bind --json example.com

# Multiple domains
pork2bind --json example.com other.com

# Save to specific directory
pork2bind --json example.com --to ./dns-cache
```

This creates `example.com.json` with all DNS records from Porkbun.

### Phase 2: Generate BIND Zone Files

Convert JSON files to BIND zone format:

```bash
# Single domain (reads example.com.json, creates db.example.com)
pork2bind --bind example.com

# Multiple domains
pork2bind --bind example.com other.com

# Specify input/output directories
pork2bind --bind example.com --from ./dns-cache --to /etc/bind/zones

# Include NS records (normally filtered out)
pork2bind --bind example.com --include-ns
```

## Command Line Options

### Global Options
- `--to <directory>` - Output directory (default: current directory)
- `--quiet` - Minimal output, only errors and warnings
- `--stdout` - Output data to stdout instead of files (for piping)
- `--help` - Show help message

### JSON Mode (`--json`)
- Fetches DNS records from Porkbun API
- Saves records as `domain.json` files (or to stdout with `--stdout`)
- Requires valid API credentials in `.env`

### BIND Mode (`--bind`)
- `--from <directory>` - Input directory for JSON files (default: current directory)
- `--include-ns` - Include NS records in output (default: filtered out)
- Converts JSON to BIND zone files as `db.domain` (or to stdout with `--stdout`)

## Examples

### Simple Workflow
```bash
# Fetch data
pork2bind --json mysite.com

# Generate zone file
pork2bind --bind mysite.com
```

### Organized Workflow
```bash
# Fetch multiple domains to cache directory
pork2bind --json site1.com site2.com site3.com --to ./dns-cache

# Convert to BIND zones in production directory
pork2bind --bind site1.com site2.com site3.com \
  --from ./dns-cache \
  --to /etc/bind/zones
```

### Including External Nameservers
```bash
# Keep Porkbun NS records in zone file
pork2bind --bind mysite.com --include-ns
```

### Output Control and Scripting
```bash
# Quiet operation (minimal output)
pork2bind --json mysite.com --quiet

# Output to stdout for piping/redirection
pork2bind --json mysite.com --stdout > mysite.json
pork2bind --bind mysite.com --stdout > db.mysite

# Combine for silent piping
pork2bind --bind mysite.com --stdout --quiet > /etc/bind/zones/db.mysite
```

### Shell Pipeline Examples
```bash
# Fetch and convert in one pipeline
pork2bind --json mysite.com --stdout | \
  jq '.[] | select(.type == "A")' | \
  pork2bind --bind mysite.com --from /dev/stdin --stdout > only-a-records.zone

# Backup all domains
for domain in $(cat domains.txt); do
  pork2bind --json $domain --stdout --quiet > backup/${domain}.json
done
```

## Output Format

### JSON Files
Raw DNS records from Porkbun API:
```json
[
  {
    "id": "123456789",
    "name": "www",
    "type": "A",
    "content": "192.0.2.1",
    "ttl": "600"
  }
]
```

### BIND Zone Files
Standard BIND format with SOA record and filtered records:
```
$TTL 3600
$ORIGIN example.com.
@    IN    SOA    ns1.example.com. admin.example.com. (
                    20250619143022  ; Serial
                    3600            ; Refresh
                    1800            ; Retry
                    1209600         ; Expire
                    3600 )          ; Minimum TTL

@                    600     IN A        192.0.2.1
www                  600     IN A        192.0.2.1
```

## Output Control

### Default Behavior
- Progress messages go to `stderr` 
- Data files saved to disk
- Full verbose output showing what's happening

### Quiet Mode (`--quiet`)
- Suppresses progress messages
- Only shows errors and warnings
- Perfect for automated scripts

### Stdout Mode (`--stdout`)
- Outputs data to `stdout` instead of files
- Progress messages still go to `stderr`
- Enables piping and shell redirection
- Can be combined with `--quiet` for pure data output

### Unix-Style Output
Following standard Unix conventions:
- **Data** → `stdout` (when using `--stdout`)
- **Progress/Status** → `stderr` 
- **Errors** → `stderr`

This allows clean piping without interference from status messages.

## Error Handling

The tool provides clear error messages for common issues:
- Missing or invalid API credentials
- Network connectivity problems
- Missing JSON files
- Permission errors
- Malformed JSON data

## Default Filtering

By default, `--bind` mode filters out NS records because:
- External NS records (like Porkbun's) shouldn't be in local BIND zones
- Prevents DNS configuration conflicts
- Most users want local DNS serving without external dependencies

Use `--include-ns` if you need the original NS records for secondary DNS setups.

## Use Cases

- **Local DNS serving**: Run your own authoritative DNS server
- **DNS backup**: Keep local copies of your DNS configuration
- **Zone file management**: Convert API data to standard BIND format
- **DNS migration**: Move from Porkbun to self-hosted DNS
- **Development**: Test DNS changes locally before deploying
- **Automation**: Script DNS updates and deployments
- **Integration**: Pipe DNS data through shell tools for processing

## Requirements

- Node.js 14.0.0 or higher
- Valid Porkbun account with API access
- Domains managed through Porkbun

## License

GPL-3.0 License - see [LICENSE](LICENSE) file for details.

## Author

Shawn Murphy <smurp@smurp.com>

## Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Add tests if applicable
5. Submit a pull request

## Support

- Report bugs: [GitHub Issues](https://github.com/smurp/pork2bind/issues)
- Documentation: This README and `--help` output
- API Documentation: [Porkbun API Docs](https://porkbun.com/api/json/v3/documentation)

## Changelog

### v1.0.0
- Initial release
- JSON and BIND conversion modes
- Automatic NS record filtering
- Batch domain processing
- Flexible directory management
- Output control with `--quiet` and `--stdout` flags
- Unix-style stdout/stderr separation for scripting
