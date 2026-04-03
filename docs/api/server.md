# Server API

The Server type is the core runtime for exposing tools, resources, and prompts over MCP.

## Constructor

### Server.init

pub fn init(config: ServerConfig) Server

Creates a new server instance.

Important ServerConfig fields:

| Field | Type | Description |
| --- | --- | --- |
| name | []const u8 | Required server name |
| version | []const u8 | Required server version |
| allocator | std.mem.Allocator | Memory allocator |
| title | ?[]const u8 | Optional title |
| description | ?[]const u8 | Optional description |
| instructions | ?[]const u8 | Optional MCP server instructions |

## Lifecycle

### Server.deinit

pub fn deinit(self: *Server) void

Releases internal maps and transport resources.

### Server.run

pub fn run(self: *Server, options: RunOptions) !void

Run options:

| Option | Description |
| --- | --- |
| .stdio | Line-delimited JSON-RPC over stdin/stdout |
| .{ .http = .{ .host = "localhost", .port = 8080 } } | HTTP listener with JSON-RPC POST endpoint |

Examples:

try server.run(.stdio);

try server.run(.{ .http = .{ .host = "api.example.com", .port = 8443 } });

## Registration Methods

### Server.addTool

pub fn addTool(self: *Server, tool: Tool) !void

### Server.addResource

pub fn addResource(self: *Server, resource: Resource) !void

### Server.addResourceTemplate

pub fn addResourceTemplate(self: *Server, template: ResourceTemplate) !void

### Server.addPrompt

pub fn addPrompt(self: *Server, prompt: Prompt) !void

## Capability Toggles

| Method | Purpose |
| --- | --- |
| enableLogging | Enables logging capability |
| enableCompletions | Enables completion capability |
| enableTasks | Enables task-augmented tool call support |

Tools/resources/prompts capabilities are enabled when you register those components.

## Notification Methods

| Method | Purpose |
| --- | --- |
| notifyToolsChanged | Emit tools/list_changed |
| notifyResourcesChanged | Emit resources/list_changed |
| notifyResourceUpdated | Emit resources/updated for a URI |
| notifyPromptsChanged | Emit prompts/list_changed |
| sendLogMessage | Emit notifications/message |
| sendProgress | Emit notifications/progress |

## Minimal Example

const std = @import("std");
const mcp = @import("mcp");

pub fn main() void {
    run() catch |err| mcp.reportError(err);
}

fn run() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();

    var server = mcp.Server.init(.{
        .name = "api-example-server",
        .version = "1.0.0",
        .allocator = gpa.allocator(),
    });
    defer server.deinit();

    try server.addTool(.{
        .name = "ping",
        .description = "Returns pong",
        .handler = struct {
            fn handler(allocator: std.mem.Allocator, _: ?std.json.Value) mcp.tools.ToolError!mcp.tools.ToolResult {
                return mcp.tools.textResult(allocator, "pong") catch return mcp.tools.ToolError.OutOfMemory;
            }
        }.handler,
    });

    server.enableLogging();
    try server.run(.stdio);
}
