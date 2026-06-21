 ```diff
--- a/tools/openapi_mock.lua
+++ b/tools/openapi_mock.lua
@@ -1,15 +1,28 @@
 #!/usr/bin/env lua5.4
 
+-- openapi_mock.lua
+-- Mock server with OpenAPI contract validation
+
 local http = require("http")
 local yaml = require("yaml")
+local io = require("io")
+local os = require("os")
+local string = require("string")
+local table = require("table")
+local math = require("math")
+local debug = require("debug")
 
 -- Configuration
 local PORT = os.getenv("MOCK_PORT") or 8080
 local OPENAPI_PATH = os.getenv("OPENAPI_PATH") or "docs/openapi/v3.yaml"
 local MOCKS_DIR = os.getenv("MOCKS_DIR") or "tools/mocks"
 
+-- CLI argument parsing
+local ALLOW_INVALID_MOCKS = false
+for _, arg in ipairs(arg or {}) do
+    if arg == "--allow-invalid-mocks" then
+        ALLOW_INVALID_MOCKS = true
+    end
+end
+
 -- Simple YAML parser wrapper
 local function load_yaml(path)
     local f, err = io.open(path, "r")
@@ -25,6 +38,7 @@
 -- Load OpenAPI spec
 local spec = load_yaml(OPENAPI_PATH)
 local paths = spec.paths or {}
+local components = spec.components or {}
 
 -- In-memory mock store: path -> method -> response
 local mocks = {}
@@ -32,6 +46,7 @@
 -- Validation state
 local validated_endpoints = {}
 local skipped_endpoints = {}
+local invalid_endpoints = {}
 
 -- Simple JSON encoder for basic types
 local function encode_json(obj)
@@ -80,6 +95,7 @@
     return result
 end
 
+-- Decode JSON string to table
 local function decode_json(str)
     -- Very basic JSON decoder for mock responses
     -- In production, use a proper JSON library
@@ -88,6 +104,7 @@
     return nil
 end
 
+-- Parse a simple JSON string into a Lua table
 local function parse_json(json_str)
     if not json_str or json_str == "" then
         return nil
@@ -97,6 +114,7 @@
     return nil
 end
 
+-- Get nested value from a table using dot-separated path
 local function get_nested(obj, path)
     if not obj then return nil end
     local parts = {}
@@ -112,6 +130,7 @@
     return current
 end
 
+-- Set nested value in a table using dot-separated path
 local function set_nested(obj, path, value)
     local parts = {}
     for part in path:gmatch("[^%.]+") do
@@ -130,6 +149,7 @@
     current[last] = value
 end
 
+-- Check if a table is an array (list)
 local function is_array(t)
     if type(t) ~= "table" then return false end
     local i = 0
@@ -140,6 +160,7 @@
     return true
 end
 
+-- Deep copy a table
 local function deep_copy(orig)
     if type(orig) ~= "table" then return orig end
     local copy = {}
@@ -149,6 +170,7 @@
     return copy
 end
 
+-- Merge two tables (shallow merge for schemas)
 local function merge_tables(a, b)
     local result = deep_copy(a)
     for k, v in pairs(b or {}) do
@@ -157,6 +179,7 @@
     return result
 end
 
+-- Resolve a $ref reference in the spec
 local function resolve_ref(ref)
     if not ref or not ref:startswith("$ref") then
         return ref
@@ -183,6 +206,7 @@
     return current
 end
 
+-- Get the schema for a response from the OpenAPI spec
 local function get_response_schema(path, method, status_code)
     local path_item = paths[path]
     if not path_item then return nil end
@@ -208,6 +232,7 @@
     return nil
 end
 
+-- Validate a value against a JSON schema
 local function validate_value(value, schema, path_str)
     local errors = {}
     path_str = path_str or "root"
@@ -336,6 +361,7 @@
     return errors
 end
 
+-- Validate a response body against the OpenAPI schema
 local function validate_response(path, method, status_code, body)
     local schema = get_response_schema(path, method, status_code)
     if not schema then
@@ -349,6 +375,7 @@
     return validate_value(body, schema, "response")
 end
 
+-- Load mock files from the mocks directory
 local function load_mocks()
     local cmd = "ls " .. MOCKS_DIR .. " 2>/dev/null"
     local handle = io.popen(cmd)
@@ -388,6 +415,7 @@
     end
 end
 
+-- Find a mock response for a given path and method
 local function find_mock(path, method)
     -- Normalize method to uppercase
     method = method:upper()
@@ -410,6 +438,7 @@
     return nil
 end
 
+-- Start the HTTP server
 local function start_server()
     local server = http.create_server("0.0.0.0", PORT, function(req, res)
         local path = req.path
@@ -449,6 +478,7 @@
     return server
 end
 
+-- Print startup summary
 local function print_summary()
     print("\n" .. string.rep("=", 60))
     print("OpenAPI Mock Server - Startup Summary")
@@ -469,6 +499,14 @@
         print("  " .. endpoint)
     end
     
+    if #invalid_endpoints > 0 then
+        print("\nInvalid Endpoints (schema violations):")
+        for _, endpoint in ipairs(invalid_endpoints) do
+            print("  " .. endpoint)
+        end
+        print("\nERROR: Mock responses violate required schema fields.")
+        print("Use --allow-invalid-mocks to bypass (development only).")
+    end
+    
     print("\n" .. string.rep("=", 60))
 end
 
@@ -478,6 +516,7 @@
     -- Load all mock files
     load_mocks()
     
+    -- Validate each loaded mock against the OpenAPI schema
     for path, methods in pairs(mocks) do
         for method, response in pairs(methods) do
             local status_code = response.status or 200
@@ -487,14 +526,35 @@
             if #errors == 0 then
                 table.insert(validated_endpoints, path .. " " .. method)
             else
-                table.insert(skipped_endpoints, path .. " " .. method .. " - " .. table.concat(errors, "; "))
+                local error_msg = path .. " " .. method .. " - " .. table.concat(errors, "; ")
+                table.insert(in