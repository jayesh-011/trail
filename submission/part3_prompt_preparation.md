# Part 3: Prompt Preparation

## Selected PR: beetbox/beets #3877 - Add readonly configuration option for web plugin

---

## 3.1.1 Repository Context (200-300 words)

Beets is a sophisticated media library management system specifically designed for music collectors who value accurate metadata and organized collections. The project serves as a command-line tool and library that helps users automatically catalog their music, correct metadata tags, fetch album artwork, and maintain clean file organization. At its core, beets acts as an intelligent intermediary between a user's music files and various music metadata databases like MusicBrainz and Discogs.

The intended users are music enthusiasts and audiophiles who maintain large personal music collections and want systematic, automated management rather than manual organization. These users typically have hundreds or thousands of albums and need tools that can handle bulk operations, auto-tagging, and consistent file naming conventions. Beets appeals particularly to users comfortable with command-line interfaces and configuration files, though it also provides a web-based interface for browsing and searching the library.

The problem domain beets addresses is digital music library management and metadata accuracy. In the digital music landscape, files often have inconsistent, incorrect, or missing metadata tags. Album art may be missing or low quality. File naming schemes vary wildly between sources. Beets solves these problems by automatically querying authoritative music databases, presenting users with matching options, and applying corrections systematically. The web plugin specifically addresses the need to browse and query the music library through a web interface, making the collection accessible via any web browser and potentially enabling remote access to the music library.

---

## 3.1.2 Pull Request Description (200-300 words)

This pull request introduces a security enhancement to the beets web plugin by adding a new configuration option called `readonly`. The specific change adds a boolean flag that controls whether write operations (HTTP DELETE and PATCH requests) are permitted through the web interface.

Before this PR, the web plugin allowed any client accessing the web interface to delete items or albums from the library, or modify their metadata, without any authentication or authorization checks. This created a significant security vulnerability, especially for users who exposed their beets web interface to a network or the internet. Any malicious actor or even accidental actions could result in permanent data loss or corruption of the music library metadata.

The changes introduce defensive checks at the beginning of every DELETE and PATCH endpoint handler. When `readonly` is set to `True` (which is the new default), these endpoints immediately return an HTTP 405 Method Not Allowed response, preventing any modifications. Users who need write functionality must explicitly opt-in by setting `readonly: no` in their beets configuration file.

The previous behavior was to allow all operations unconditionally, which meant the web interface was inherently unsafe to expose without additional external security measures like reverse proxy authentication or firewall rules. The new behavior makes the interface safe by default, following the security principle of least privilege. Users can still query, search, and browse their entire library when in readonly mode, but deletion and modification require explicit configuration.

This is considered a breaking change because existing users who relied on DELETE or PATCH operations will need to update their configuration files. However, the maintainers determined that security should take precedence over backward compatibility in this case, as the previous behavior posed genuine risk of data loss.

---

## 3.1.3 Acceptance Criteria (Minimum 5 criteria)

✓ **AC1:** When the web plugin is initialized without a `readonly` configuration specified, the system should default to `readonly: True`, preventing all write operations.

✓ **AC2:** When `readonly: True` is set in the configuration, any HTTP DELETE request to remove an item or album should return HTTP 405 (Method Not Allowed) status code with an appropriate error message, and no database modifications should occur.

✓ **AC3:** When `readonly: True` is set in the configuration, any HTTP PATCH request to update item or album metadata should return HTTP 405 (Method Not Allowed) status code with an appropriate error message, and no database modifications should occur.

✓ **AC4:** When `readonly: False` is explicitly set in the configuration, HTTP DELETE requests should successfully remove items or albums from the library and return appropriate success responses (HTTP 200 or 204).

✓ **AC5:** When `readonly: False` is explicitly set in the configuration, HTTP PATCH requests should successfully update item or album metadata in the library and return appropriate success responses (HTTP 200).

✓ **AC6:** All HTTP GET requests (query, search, browse operations) should function identically regardless of the `readonly` setting, maintaining full read access to the library.

✓ **AC7:** The Flask application configuration should correctly read the `readonly` value from the beets configuration file during plugin initialization and store it in `app.config['READONLY']`.

✓ **AC8:** Error responses for rejected write operations should include clear messaging indicating that the operation is not allowed due to readonly mode, helping users understand why their request failed.

---

## 3.1.4 Edge Cases (Minimum 3 cases)

**Edge Case 1: Configuration File Parsing**
- **Scenario:** User specifies `readonly` with non-boolean values (e.g., "yes", "1", "enabled")
- **Expected Behavior:** The configuration parser should handle various truthy/falsy representations correctly, converting strings like "yes", "true", "1" to boolean `True`, and "no", "false", "0" to boolean `False`
- **Risk:** Configuration misinterpretation could leave the interface vulnerable or unexpectedly locked down

**Edge Case 2: Mid-Request Configuration Changes**
- **Scenario:** The configuration file is modified while the web server is running (changing `readonly` from False to True or vice versa)
- **Expected Behavior:** The system should either require a restart to pick up configuration changes, or implement safe configuration reloading that doesn't affect in-flight requests
- **Risk:** Race conditions where a write operation is partially completed when the configuration changes

**Edge Case 3: Plugin Not Configured**
- **Scenario:** User has the web plugin enabled but doesn't have any configuration for it in their config file
- **Expected Behavior:** The plugin should use sensible defaults including `readonly: True`, ensuring security even with minimal configuration
- **Risk:** Unconfigured installations could be vulnerable if defaults aren't security-conscious

**Edge Case 4: Malformed Write Requests**
- **Scenario:** Client sends a DELETE or PATCH request with malformed data or invalid item IDs when `readonly: False`
- **Expected Behavior:** Even with write operations enabled, the system should validate inputs and return appropriate error codes (400 Bad Request, 404 Not Found) rather than causing database errors
- **Risk:** Invalid requests could cause server errors or inconsistent database state

**Edge Case 5: Concurrent Write Operations**
- **Scenario:** Multiple DELETE or PATCH requests target the same item simultaneously when `readonly: False`
- **Expected Behavior:** The database layer should handle concurrent modifications gracefully, using transactions or locks to prevent race conditions
- **Risk:** Concurrent modifications could lead to lost updates or database inconsistencies

---

## 3.1.5 Initial Prompt (300-500 words)

**Task: Implement readonly configuration option for beets web plugin**

You are working on the beetbox/beets repository, which is a Python-based music library management system. Your task is to add a security feature to the web plugin that prevents unauthorized modifications to the music library through the web interface.

**Background:**
The beets web plugin provides a Flask-based HTTP API for querying and managing music libraries. Currently, the plugin allows DELETE operations (to remove items/albums) and PATCH operations (to update metadata) without any access controls, which creates a security vulnerability. Users who expose their web interface to a network risk having their library modified or deleted by unauthorized parties.

**Your Objective:**
Implement a new configuration option called `readonly` that defaults to `True`. When enabled, this flag should prevent all write operations (DELETE and PATCH) while allowing read operations (GET) to function normally. Users who need write functionality must explicitly set `readonly: no` in their configuration.

**Specific Requirements:**

1. **Configuration Implementation:**
   - Add a `readonly` configuration option to the web plugin with a default value of `True`
   - Ensure the configuration is properly read from the beets config file and stored in Flask's `app.config['READONLY']`
   - The configuration should support both boolean values and string representations ("yes"/"no", "true"/"false")

2. **Endpoint Modifications:**
   - Modify all DELETE endpoint handlers to check `app.config['READONLY']` before processing
   - Modify all PATCH endpoint handlers to check `app.config['READONLY']` before processing
   - When `readonly` is enabled, return HTTP 405 (Method Not Allowed) with a clear error message
   - Ensure GET endpoints remain unaffected and continue working regardless of the `readonly` setting

3. **Documentation:**
   - Update the web plugin documentation (docs/plugins/web.rst) to explain the new `readonly` option
   - Add a changelog entry (docs/changelog.rst) noting this as a breaking change
   - Provide clear examples showing how to enable write operations when needed

4. **Testing:**
   - Add test cases verifying DELETE operations return 405 when `readonly: True`
   - Add test cases verifying DELETE operations succeed when `readonly: False`
   - Add test cases verifying PATCH operations return 405 when `readonly: True`
   - Add test cases verifying PATCH operations succeed when `readonly: False`
   - Add test cases confirming GET operations work in both modes
   - Add tests for configuration parsing and default values

**Important Considerations:**
- Reference the acceptance criteria provided: defaults must be secure, error responses must be clear, and all read operations must remain functional
- Consider the edge cases: configuration parsing variations, malformed requests, and concurrent operations
- This is a breaking change - document it clearly so users understand they need to update their configuration to restore write functionality
- Follow existing code patterns in the beets codebase, particularly the Flask app structure used by the web plugin
- Ensure backward compatibility for read operations to avoid disrupting existing workflows

**Testing Requirements:**
After implementation, run the web plugin test suite to ensure all new and existing tests pass. Verify that the readonly mode actually prevents modifications by attempting DELETE and PATCH operations with the default configuration.

---

## Integrity Declaration

I declare that all written content in this assessment is my own work, created without the use of AI language models or automated writing tools. All technical analysis and documentation reflects my personal understanding and has been written in my own words.
