# GitHub Copilot Instructions for Spine

## Project Overview

Spine is a high-performance, multi-threaded SNMP poller written in C for the Cacti network monitoring system. It replaces the PHP-based `cmd.php` poller with a faster native implementation that leverages pthreads, Net-SNMP, and MySQL/MariaDB.

## Priority Guidelines

When generating code for this repository:

1. **Version Compatibility**: This is a C project using GNU Autotools (Autoconf 2.63+, Automake 1.13+). Never use C99+ features without conditional compilation guards.
2. **Context Files**: Prioritize patterns defined in this `.github/copilot` directory.
3. **Codebase Patterns**: Match existing code style exactly—this is a mature, production codebase.
4. **Architectural Consistency**: Maintain the monolithic, multi-threaded architecture with clear module separation.
5. **Code Quality**: Prioritize performance, security, and thread safety in all generated code.

## Technology Stack

### Language & Standards
- **Language**: C (ANSI C with some C99 features via conditional compilation)
- **Build System**: GNU Autotools (autoconf, automake, libtool)
- **Threading**: POSIX Threads (pthreads)

### Required Dependencies
| Library | Purpose |
|---------|---------|
| `libmysqlclient` / `libmariadb` | Database connectivity |
| `libnetsnmp` | SNMP polling operations |
| `libpthread` | Multi-threading |
| `libssl` / `libcrypto` | SSL/TLS support |
| `libm` | Math functions |
| `libdl` | Dynamic loading |
| `libz` | Compression (MySQL requirement) |

### Version Information
- **Spine Version**: 1.3.0 (from `configure.ac`)
- **License**: GNU LGPL v2.1
- **Copyright**: 2004-2024 The Cacti Group

## Code Style & Formatting

### File Header Template
Every source file MUST include this exact header format:
```c
/*
 ex: set tabstop=4 shiftwidth=4 autoindent:
 +-------------------------------------------------------------------------+
 | Copyright (C) 2004-2024 The Cacti Group                                 |
 |                                                                         |
 | This program is free software; you can redistribute it and/or           |
 | modify it under the terms of the GNU Lesser General Public              |
 | License as published by the Free Software Foundation; either            |
 | version 2.1 of the License, or (at your option) any later version.      |
 |                                                                         |
 | This program is distributed in the hope that it will be useful,         |
 | but WITHOUT ANY WARRANTY; without even the implied warranty of          |
 | MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the           |
 | GNU Lesser General Public License for more details.                     |
 |                                                                         |
 | You should have received a copy of the GNU Lesser General Public        |
 | License along with this library; if not, write to the Free Software     |
 | Foundation, Inc., 51 Franklin Street, Fifth Floor, Boston, MA           |
 | 02110-1301, USA                                                         |
 |                                                                         |
 +-------------------------------------------------------------------------+
 | spine: a backend data gatherer for cacti                                |
 +-------------------------------------------------------------------------+
 | This poller would not have been possible without:                       |
 |   - Larry Adams (current development and enhancements)                  |
 |   - Rivo Nurges (rrd support, mysql poller cache, misc functions)       |
 |   - RTG (core poller code, pthreads, snmp, autoconf examples)           |
 |   - Brady Alleman/Doug Warner (threading ideas, implementation details) |
 +-------------------------------------------------------------------------+
 | - Cacti - http://www.cacti.net/                                         |
 +-------------------------------------------------------------------------+
*/
```

### Indentation & Whitespace
- **Tab size**: 4 spaces (use tabs, not spaces)
- **Shift width**: 4
- **Autoindent**: enabled
- Vim modeline at top of file: `ex: set tabstop=4 shiftwidth=4 autoindent:`

### Naming Conventions

| Element | Convention | Examples |
|---------|------------|----------|
| Functions | `snake_case` | `db_connect()`, `poll_host()`, `snmp_host_init()` |
| Variables | `snake_case` | `host_id`, `error_count`, `mysql_row` |
| Constants/Macros | `UPPER_SNAKE_CASE` | `BUFSIZE`, `MAX_THREADS`, `LOCK_SNMP` |
| Struct types | `snake_case_t` | `host_t`, `ping_t`, `config_t`, `pool_t` |
| Header guards | `_FILENAME_H_` | `_SPINE_H_`, `SPINE_COMMON_H` |

### Buffer Size Constants
Use these predefined buffer sizes consistently:
```c
#define TINY_BUFSIZE 16
#define SMALL_BUFSIZE 256
#define MEDIUM_BUFSIZE 512
#define BUFSIZE 1024
#define DBL_BUFSIZE 2048
#define LRG_BUFSIZE 8096
#define BIG_BUFSIZE 65535
#define MEGA_BUFSIZE 1024000
#define HUGE_BUFSIZE 2048000
```

## Architectural Patterns

### Module Organization
The codebase is organized into focused modules:

| File | Responsibility |
|------|----------------|
| `spine.c` | Main entry point, thread spawning, command-line parsing |
| `poller.c` | Core polling logic, host processing |
| `snmp.c` | SNMP session management, queries |
| `sql.c` | MySQL/MariaDB connection pool, queries |
| `ping.c` | ICMP, TCP, UDP ping implementations |
| `php.c` | PHP Script Server communication |
| `util.c` | Utility functions, configuration reading |
| `locks.c` | Thread synchronization primitives |
| `error.c` | Signal handling |
| `keywords.c` | Keyword parsing |
| `nft_popen.c` | Custom popen implementation |

### Header Structure
- `common.h` — Central include file with all system headers and library includes
- `spine.h` — Main definitions, constants, macros, and struct definitions
- `<module>.h` — Function prototypes for each module

### Include Pattern
Always include headers in this order:
```c
#include "common.h"
#include "spine.h"
// Additional module-specific headers if needed
```

## Thread Safety Patterns

### Mutex Locking
Use the established locking infrastructure:
```c
// Lock acquisition
thread_mutex_lock(LOCK_NAME);

// Critical section code

// Lock release
thread_mutex_unlock(LOCK_NAME);
```

### Available Locks
```c
LOCK_SNMP        // SNMP operations
LOCK_SETEUID     // setuid operations
LOCK_GHBN        // gethostbyname (deprecated, use getaddrinfo)
LOCK_POOL        // Database connection pool
LOCK_PHP         // PHP script server selection
LOCK_PHP_PROC_N  // Individual PHP processes (0-14)
LOCK_THDET       // Thread details
LOCK_HOST_TIME   // Host timing data
```

### Thread Cleanup Pattern
Use pthread cleanup handlers:
```c
void *child(void *arg) {
    pthread_cleanup_push(child_cleanup, arg);
    
    // Thread work here
    
    pthread_cleanup_pop(1);
    pthread_exit(0);
    exit(0);
}
```

## Logging Patterns

### Logging Macros
Use level-appropriate logging macros with double parentheses:
```c
SPINE_LOG(("ERROR: Message with %s", variable));           // Always logged
SPINE_LOG_LOW(("INFO: Low priority message"));             // VERBOSITY >= LOW
SPINE_LOG_MEDIUM(("INFO: Medium priority message"));       // VERBOSITY >= MEDIUM
SPINE_LOG_HIGH(("DEBUG: High priority message"));          // VERBOSITY >= HIGH
SPINE_LOG_DEBUG(("DEBUG: Debug message"));                 // VERBOSITY >= DEBUG
SPINE_LOG_DEVDBG(("DEVDBG: Developer debug message"));     // VERBOSITY >= DEVDBG
```

### Log Message Format
```c
// Include context identifiers
SPINE_LOG(("Device[%i] ERROR: Description", host_id));
SPINE_LOG(("Device[%i] HT[%i] DEBUG: Description", host_id, host_thread));
SPINE_LOG(("SS[%i] ERROR: Script Server error", php_process));
```

### Verbosity Levels
```c
POLLER_VERBOSITY_NONE   = 1
POLLER_VERBOSITY_LOW    = 2
POLLER_VERBOSITY_MEDIUM = 3
POLLER_VERBOSITY_HIGH   = 4
POLLER_VERBOSITY_DEBUG  = 5
POLLER_VERBOSITY_DEVDBG = 6
```

## Error Handling Patterns

### Fatal Errors
Use `die()` for unrecoverable errors:
```c
die("FATAL: malloc() failed during strdup() for %s", reason);
die("ERROR: Fatal malloc error: poller.c host struct!");
```

### Memory Allocation Pattern
Always check allocations and use the `STRDUP_OR_DIE` macro:
```c
// Manual allocation
if (!(host = (host_t *) malloc(sizeof(host_t)))) {
    die("ERROR: Fatal malloc error: poller.c host struct!");
}
memset(host, 0, sizeof(host_t));

// String duplication
STRDUP_OR_DIE(hostname, set.db_host, "db_host")
```

### Memory Cleanup
Use the `SPINE_FREE` macro:
```c
SPINE_FREE(pointer);  // Frees and sets to NULL
```

### Database Error Handling
Handle MySQL errors with retry logic:
```c
error = mysql_errno(mysql);

if (error == 2013 || error == 2006) {  // Connection lost
    db_reconnect(mysql, error, "function_name");
    continue;
}

if (error == 1213 || error == 1205) {  // Deadlock
    usleep(50000);
    error_count++;
    if (error_count > 30) {
        SPINE_LOG(("ERROR: Too many Lock/Deadlock errors!"));
        return FALSE;
    }
    continue;
}
```

## Function Documentation Style

Use Doxygen-style comments for functions:
```c
/*! \fn void function_name(int param1, char *param2)
 *  \brief Short description of what the function does.
 *  \param param1 Description of first parameter
 *  \param param2 Description of second parameter
 *
 *  Detailed description of the function's behavior,
 *  including any important notes about usage.
 *
 *  \return Description of return value
 *
 */
```

## Database Patterns

### Connection Pool Usage
```c
pool_t *local_cnn = NULL;
MYSQL mysql;

local_cnn = db_get_connection(LOCAL);
mysql = local_cnn->mysql;

// Use mysql connection

db_release_connection(LOCAL, local_cnn->id);
```

### Query Execution
```c
// SELECT queries
MYSQL_RES *result;
MYSQL_ROW row;

result = db_query(&mysql, LOCAL, query);
if (result != NULL) {
    while ((row = mysql_fetch_row(result))) {
        // Process row
    }
    db_free_result(result);
}

// INSERT/UPDATE queries
if (db_insert(&mysql, LOCAL, query) == FALSE) {
    SPINE_LOG(("ERROR: Query failed"));
}
```

### SQL String Escaping
```c
char escaped[BIG_BUFSIZE];
db_escape(&mysql, escaped, sizeof(escaped), input_string);
```

## SNMP Patterns

### Session Initialization
```c
void *session = snmp_host_init(
    host_id,
    hostname,
    snmp_version,
    snmp_community,
    snmp_username,
    snmp_password,
    snmp_auth_protocol,
    snmp_priv_passphrase,
    snmp_priv_protocol,
    snmp_context,
    snmp_engine_id,
    snmp_port,
    snmp_timeout
);
```

### Conditional Compilation for Net-SNMP
```c
#ifdef NETSNMP_DS_LIB_DONT_PERSIST_STATE
    netsnmp_ds_set_boolean(NETSNMP_DS_LIBRARY_ID, 
                           NETSNMP_DS_LIB_DONT_PERSIST_STATE, 1);
#endif
```

## Platform Compatibility

### Conditional Compilation Guards
```c
#ifdef __CYGWIN__
    // Windows/Cygwin-specific code
#endif

#ifdef SOLAR_THREAD
    // Solaris threading model
#endif

#ifdef HAVE_LCAP
    // Linux capabilities support
#endif
```

### Unused Parameter Handling
```c
UNUSED_PARAMETER(argc);
UNUSED_VARIABLE(some_var);
```

## Security Considerations

### Input Validation
- Always use `db_escape()` for SQL string parameters
- Validate buffer sizes before string operations
- Use `snprintf()` instead of `sprintf()` for bounded writes

### String Operations
```c
// Safe string copy with size limit
strncopy(dst, src, sizeof(dst));

// Use project macro for struct member copies
STRNCOPY(host->hostname, row[1]);
```

### Privilege Dropping
```c
#ifdef HAVE_LCAP
void drop_root(uid_t server_uid, gid_t server_gid);
#endif
```

## Build System

### Adding New Source Files
1. Add to `spine_SOURCES` in `Makefile.am`
2. Run `./bootstrap` to regenerate build files
3. Run `./configure && make`

### Configure Options
```bash
./configure --help                    # Show all options
./configure --with-mysql=/path        # MySQL location
./configure --with-snmp=/path         # Net-SNMP location
./configure --enable-lcap             # Linux capabilities
./configure --with-results-buffer=N   # Results buffer size
./configure --with-max-scripts=N      # Max simultaneous scripts
```

## Testing & Debugging

### Debug Execution
```bash
./spine -R -S -V 5   # Read-only, stdout, debug verbosity
./spine -C /path/to/spine.conf -f 1 -l 10  # Poll hosts 1-10
```

### Debug Device Logging
Use `is_debug_device(host_id)` to enable verbose logging for specific devices.

## Version Control Guidelines

- Follow Semantic Versioning (MAJOR.MINOR.PATCH)
- Update `CHANGELOG` with each release
- Format: `issue#NNN:` or `feature#NNN:` prefix for changes

## Common Patterns to Follow

### Boolean Values
```c
#define FALSE 0
#define TRUE 1
```

### Return Patterns
- Return `TRUE`/`FALSE` for success/failure
- Return `NULL` or `0` for pointer/handle failures
- Use negative values for error codes when appropriate

### Assertion Usage
```c
assert(psql != 0);
assert(setting != 0);
```

## What NOT to Do

1. **Never** use C11/C17 features without guards
2. **Never** use `sprintf()` for user-controlled input
3. **Never** allocate memory without checking the return
4. **Never** access shared data without appropriate locking
5. **Never** ignore MySQL error codes
6. **Never** hardcode buffer sizes—use defined constants
7. **Never** use `gethostbyname()`—use `getaddrinfo()` instead
8. **Never** create global variables without `extern` declarations in headers
