# Slack MCP Server

### Note about changes from upstream

Note: The purpose of this repository from the authors POV was to experiment with combining an LLM with a read-only view of Slack from the same point of view as a human client **without administrative access to the Slack workspace**. This means bot authentication (e.g. `xoxb-` tokens) is out of the picture.

A forked version of [github.com/slack-go/slack](https://github.com/slack-go/slack) is used to access the Slack API in order to send custom requests to non-public APIs to speed up development and channel cache updates. Some additional tweaks were made to the [Slack MCP Server used](https://github.com/korotovsky/slack-mcp-server) to expose
this as a tool, and some fixes were made to the original codebase for the channel caching and channel lookup (which may be unnecessary given the channel caching fixes).

**Warning**: The use of these non-public APIs may trigger Slack reacting to the authentication and API usage as a third-party client and reacting to it as suspicous activity. For the author, this has happened once over a couple of weeks and required signing in again. Use at your own risk.

Small note about caches:

- There's two main caches: the users cache and the channels cache.
- The users cache is small and quick to populate. It allows lookup between the UXXXXXX and the @username formats, which is required for searches/lookups etc.
- The channels cache is larger but very slow to populate on larger workspaces, as it requires paginating through all channels, and because the pagination size includes filtered out channels (e.g due to archivals etc), you may request 1000 channels per page only to end up with 2-6. This leads straight into rate limiting which makes the process very very slow. It's also required to look up channels between the CXxxxxxx and #channelname formats.

To work with this there's three relevant flags:

- `--enable-users-cache=true/false`: Defaults to true. Enables the users cache. Required for searches/lookups etc.
- `--enable-channels-cache=true/false`: Defaults to true. Enables the channels cache. Required for channel lookups etc. If this takes too long for your workspace, disable it and enable the next one instead:
- `--client-boot=true/false`: Defaults to false. Enables client boot. This reads the channel ids from the same bootstrap API as the Slack client, which enables the cache for the channels the current user is already in, and is a quick operation.

Once the users/channels cache files exist, they will be used without consulting the Slack API. To refresh the caches, delete the files and restart the MCP server. If client boot is enabled, it will be used during /every/ start to refresh the cache, so no deletion is required. If you're running in Docker, you may want to consider volume mounts to cache the files between multiple restarts (or not).

## Upstream readme continues here:

Model Context Protocol (MCP) server for Slack Workspaces. The most powerful MCP Slack server — supports Stdio and SSE transports, proxy settings, DMs, Group DMs, Smart History fetch (by date or count), may work via OAuth or in complete stealth mode with no permissions and scopes in Workspace 😏.

> [!IMPORTANT]  
> We need your support! Each month, over 30,000 engineers visit this repository, and more than 9,000 are already using it.
> 
> If you appreciate the work our [contributors](https://github.com/korotovsky/slack-mcp-server/graphs/contributors) have put into this project, please consider giving the repository a star.

This feature-rich Slack MCP Server has:
- **Stealth and OAuth Modes**: Run the server without requiring additional permissions or bot installations (stealth mode), or use secure OAuth tokens for access without needing to refresh or extract tokens from the browser (OAuth mode).
- **Enterprise Workspaces Support**: Possibility to integrate with Enterprise Slack setups.
- **Channel and Thread Support with `#Name` `@Lookup`**: Fetch messages from channels and threads, including activity messages, and retrieve channels using their names (e.g., #general) as well as their IDs.
- **Smart History**: Fetch messages with pagination by date (d1, 7d, 1m) or message count.
- **Search Messages**: Search messages in channels, threads, and DMs using various filters like date, user, and content.
- **Safe Message Posting**: The `conversations_add_message` tool is disabled by default for safety. Enable it via an environment variable, with optional channel restrictions.
- **DM and Group DM support**: Retrieve direct messages and group direct messages.
- **Embedded user information**: Embed user information in messages, for better context.
- **Cache support**: Cache users and channels for faster access.
- **Stdio/SSE Transports & Proxy Support**: Use the server with any MCP client that supports Stdio or SSE transports, and configure it to route outgoing requests through a proxy if needed.

### Analytics Demo

![Analytics](images/feature-1.gif)

### Add Message Demo

![Add Message](images/feature-2.gif)

## Tools

### 1. conversations_history:
Get messages from the channel (or DM) by channel_id, the last row/column in the response is used as 'cursor' parameter for pagination if not empty
- **Parameters:**
  - `channel_id` (string, required):     - `channel_id` (string): ID of the channel in format Cxxxxxxxxxx or its name starting with `#...` or `@...` aka `#general` or `@username_dm`.
  - `include_activity_messages` (boolean, default: false): If true, the response will include activity messages such as `channel_join` or `channel_leave`. Default is boolean false.
  - `cursor` (string, optional): Cursor for pagination. Use the value of the last row and column in the response as next_cursor field returned from the previous request.
  - `limit` (string, default: "1d"): Limit of messages to fetch in format of maximum ranges of time (e.g. 1d - 1 day, 30d - 30 days, 90d - 90 days which is a default limit for free tier history) or number of messages (e.g. 50). Must be empty when 'cursor' is provided.

### 2. conversations_replies:
Get a thread of messages posted to a conversation by channelID and `thread_ts`, the last row/column in the response is used as `cursor` parameter for pagination if not empty.
- **Parameters:**
  - `channel_id` (string, required): ID of the channel in format `Cxxxxxxxxxx` or its name starting with `#...` or `@...` aka `#general` or `@username_dm`.
  - `thread_ts` (string, required): Unique identifier of either a thread’s parent message or a message in the thread. ts must be the timestamp in format `1234567890.123456` of an existing message with 0 or more replies.
  - `include_activity_messages` (boolean, default: false): If true, the response will include activity messages such as 'channel_join' or 'channel_leave'. Default is boolean false.
  - `cursor` (string, optional): Cursor for pagination. Use the value of the last row and column in the response as next_cursor field returned from the previous request.
  - `limit` (string, default: "1d"): Limit of messages to fetch in format of maximum ranges of time (e.g. 1d - 1 day, 30d - 30 days, 90d - 90 days which is a default limit for free tier history) or number of messages (e.g. 50). Must be empty when 'cursor' is provided.

### 3. conversations_add_message
Add a message to a public channel, private channel, or direct message (DM, or IM) conversation by channel_id and thread_ts.

> **Note:** Posting messages is disabled by default for safety. To enable, set the `SLACK_MCP_ADD_MESSAGE_TOOL` environment variable. If set to a comma-separated list of channel IDs, posting is enabled only for those specific channels. See the Environment Variables section below for details.

- **Parameters:**
  - `channel_id` (string, required): ID of the channel in format `Cxxxxxxxxxx` or its name starting with `#...` or `@...` aka `#general` or `@username_dm`.
  - `thread_ts` (string, optional): Unique identifier of either a thread’s parent message or a message in the thread_ts must be the timestamp in format `1234567890.123456` of an existing message with 0 or more replies. Optional, if not provided the message will be added to the channel itself, otherwise it will be added to the thread.
  - `payload` (string, required): Message payload in specified content_type format. Example: 'Hello, world!' for text/plain or '# Hello, world!' for text/markdown.
  - `content_type` (string, default: "text/markdown"): Content type of the message. Default is 'text/markdown'. Allowed values: 'text/markdown', 'text/plain'.

### 4. conversations_search_messages
Search messages in a public channel, private channel, or direct message (DM, or IM) conversation using filters. All filters are optional, if not provided then search_query is required.
- **Parameters:**
  - `search_query` (string, optional): Search query to filter messages. Example: `marketing report`.
  - `filter_in_channel` (string, optional): Filter messages in a specific channel by its ID or name. Example: `C1234567890` or `#general`. If not provided, all channels will be searched.
  - `filter_in_im_or_mpim` (string, optional): Filter messages in a direct message (DM) or multi-person direct message (MPIM) conversation by its ID or name. Example: `D1234567890` or `@username_dm`. If not provided, all DMs and MPIMs will be searched.
  - `filter_users_with` (string, optional): Filter messages with a specific user by their ID or display name in threads and DMs. Example: `U1234567890` or `@username`. If not provided, all threads and DMs will be searched.
  - `filter_users_from` (string, optional): Filter messages from a specific user by their ID or display name. Example: `U1234567890` or `@username`. If not provided, all users will be searched.
  - `filter_date_before` (string, optional): Filter messages sent before a specific date in format `YYYY-MM-DD`. Example: `2023-10-01`, `July`, `Yesterday` or `Today`. If not provided, all dates will be searched.
  - `filter_date_after` (string, optional): Filter messages sent after a specific date in format `YYYY-MM-DD`. Example: `2023-10-01`, `July`, `Yesterday` or `Today`. If not provided, all dates will be searched.
  - `filter_date_on` (string, optional): Filter messages sent on a specific date in format `YYYY-MM-DD`. Example: `2023-10-01`, `July`, `Yesterday` or `Today`. If not provided, all dates will be searched.
  - `filter_date_during` (string, optional): Filter messages sent during a specific period in format `YYYY-MM-DD`. Example: `July`, `Yesterday` or `Today`. If not provided, all dates will be searched.
  - `filter_threads_only` (boolean, default: false): If true, the response will include only messages from threads. Default is boolean false.
  - `cursor` (string, default: ""): Cursor for pagination. Use the value of the last row and column in the response as next_cursor field returned from the previous request.
  - `limit` (number, default: 20): The maximum number of items to return. Must be an integer between 1 and 100.

### 5. channels_list:
Get list of channels
- **Parameters:**
  - `channel_types` (string, required): Comma-separated channel types. Allowed values: `mpim`, `im`, `public_channel`, `private_channel`. Example: `public_channel,private_channel,im`
  - `sort` (string, optional): Type of sorting. Allowed values: `popularity` - sort by number of members/participants in each channel.
  - `limit` (number, default: 100): The maximum number of items to return. Must be an integer between 1 and 1000 (maximum 999).
  - `cursor` (string, optional): Cursor for pagination. Use the value of the last row and column in the response as next_cursor field returned from the previous request.

## Setup Guide

- [Authentication Setup](docs/01-authentication-setup.md)
- [Installation](docs/02-installation.md)
- [Configuration and Usage](docs/03-configuration-and-usage.md)

### Environment Variables (Quick Reference)

| Variable                       | Required? | Default                   | Description                                                                                                                                                                                                                                                                               |
|--------------------------------|-----------|---------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `SLACK_MCP_XOXC_TOKEN`         | Yes*      | `nil`                     | Slack browser token (`xoxc-...`)                                                                                                                                                                                                                                                          |
| `SLACK_MCP_XOXD_TOKEN`         | Yes*      | `nil`                     | Slack browser cookie `d` (`xoxd-...`)                                                                                                                                                                                                                                                     |
| `SLACK_MCP_XOXP_TOKEN`         | Yes*      | `nil`                     | User OAuth token (`xoxp-...`) — alternative to xoxc/xoxd                                                                                                                                                                                                                                  |
| `SLACK_MCP_PORT`               | No        | `13080`                   | Port for the MCP server to listen on                                                                                                                                                                                                                                                      |
| `SLACK_MCP_HOST`               | No        | `127.0.0.1`               | Host for the MCP server to listen on                                                                                                                                                                                                                                                      |
| `SLACK_MCP_SSE_API_KEY`        | No        | `nil`                     | Bearer token for SSE transport                                                                                                                                                                                                                                                            |
| `SLACK_MCP_PROXY`              | No        | `nil`                     | Proxy URL for outgoing requests                                                                                                                                                                                                                                                           |
| `SLACK_MCP_USER_AGENT`         | No        | `nil`                     | Custom User-Agent (for Enterprise Slack environments)                                                                                                                                                                                                                                     |
| `SLACK_MCP_SERVER_CA`          | No        | `nil`                     | Path to CA certificate                                                                                                                                                                                                                                                                    |
| `SLACK_MCP_SERVER_CA_INSECURE` | No        | `false`                   | Trust all insecure requests (NOT RECOMMENDED)                                                                                                                                                                                                                                             |
| `SLACK_MCP_ADD_MESSAGE_TOOL`   | No        | `nil`                     | Enable message posting via `conversations_add_message` by setting it to true for all channels, a comma-separated list of channel IDs to whitelist specific channels, or use `!` before a channel ID to allow all except specified ones, while an empty value disables posting by default. |
| `SLACK_MCP_USERS_CACHE`        | No        | `.users_cache.json`       | Path to the users cache file. Used to cache Slack user information to avoid repeated API calls on startup.                                                                                                                                                                                |
| `SLACK_MCP_CHANNELS_CACHE`     | No        | `.channels_cache_v2.json` | Path to the channels cache file. Used to cache Slack channel information to avoid repeated API calls on startup.                                                                                                                                                                          |

*You need either `xoxp` **or** both `xoxc`/`xoxd` tokens for authentication.

### Debugging Tools

```bash
# Run the inspector with stdio transport
npx @modelcontextprotocol/inspector go run mcp/mcp-server.go --transport stdio

# View logs
tail -n 20 -f ~/Library/Logs/Claude/mcp*.log
```

## Security

- Never share API tokens
- Keep .env files secure and private

## License

Licensed under MIT - see [LICENSE](LICENSE) file. This is not an official Slack product.
