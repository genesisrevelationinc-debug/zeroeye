 ```diff
--- a/tools/openapi_mock.lua
+++ b/tools/openapi_mock.lua
@@ -1,15 +1,28 @@
 #!/usr/bin/env lua
 
+-- openapi_mock.lua: Mock server with OpenAPI contract validation
+--
+-- Usage: lua tools/openapi_mock.lua [options]
+--   --allow-invalid-mocks    Allow mock responses that violate the OpenAPI schema (development only)
+--
+-- The server validates all generated mock responses against docs/openapi/v3.yaml
+-- before serving them. Invalid mocks cause startup to fail unless --allow-invalid-mocks
+-- is provided.
+
 local http = require("http")
 local yaml = require("yaml")
 
+-- Configuration
 local PORT = 8089
 local HOST = "0.0.0.0"
 local MOCK_DIR = "tools/openapi_mocks"
 local OPENAPI_SPEC_PATH = "docs/openapi/v3.yaml"
 
+-- Parse command line arguments
+local ALLOW_INVALID_MOCKS = false
+for _, arg in ipairs(arg or {}) do
+  if arg == "--allow-invalid-mocks" then
+    ALLOW_INVALID_MOCKS = true
+  end
+end
+
 -- In-memory storage
 local spec = {}
 local mocks = {}
@@ -18,6 +31,7 @@
 local function load_yaml(path)
   local file = io.open(path, "r")
   if not file then
+    print("ERROR: Cannot open file: " .. path)
     return nil
   end
   local content = file:read("*all")
@@ -26,6 +40,7 @@
   local ok, result = pcall(yaml.load, content)
   if ok then
     return result
+  else
+    print("ERROR: Failed to parse YAML: " .. tostring(result))
   end
   return nil
 end
@@ -33,6 +48,7 @@
 -- Load OpenAPI spec
 local function load_spec()
   spec = load_yaml(OPENAPI_SPEC_PATH) or {}
+  print("Loaded OpenAPI spec from " .. OPENAPI_SPEC_PATH)
 end
 
 -- Load mock responses from directory
  local function load_mocks()
@@ -40,6 +56,7 @@
   local handle = io.popen('ls "' .. MOCK_DIR .. '" 2>/dev/null')
   if not handle then
     mocks = {}
+    print("WARNING: No mock directory found at " .. MOCK_DIR)
     return
   end
   
@@ -58,6 +75,7 @@
       end
     end
   end
+  print("Loaded " .. tostring(#mocks) .. " mock files from " .. MOCK_DIR)
 end
 
 -- Simple path matching with parameter support
@@ -84,6 +102,7 @@
   return nil
 end
 
+-- Extract schema for a specific response from the OpenAPI spec
 local function get_response_schema(path, method, status_code)
   if not spec.paths then
     return nil
@@ -108,6 +127,7 @@
   return nil
 end
 
+-- Validate a value against a JSON schema-like structure
 local function validate_value(value, schema, path)
   local errors = {}
   
@@ -115,6 +135,10 @@
     return errors
   end
   
+  if type(schema) ~= "table" then
+    return errors
+  end
+  
   local schema_type = schema.type
   
   -- Check type
@@ -123,6 +147,8 @@
       table.insert(errors, path .. ": expected object, got " .. type(value))
   elseif schema_type == "array" and type(value) ~= "table" then
     table.insert(errors, path .. ": expected array, got " .. type(value))
+  elseif schema_type == "string" and type(value) ~= "string" then
+    table.insert(errors, path .. ": expected string, got " .. type(value))
   elseif schema_type == "number" and type(value) ~= "number" then
     table.insert(errors, path .. ": expected number, got " .. type(value))
   elseif schema_type == "integer" and (type(value) ~= "number" or value % 1 ~= 0) then
@@ -131,6 +157,8 @@
     table.insert(errors, path .. ": expected boolean, got " .. type(value))
   end
   
+  -- If type doesn't match and we already have an error, skip deeper validation
+  if #errors > 0 then
+    return errors
+  end
+  
   -- Check required fields
   if schema.required and type(value) == "table" then
     for _, field in ipairs(schema.required) do
@@ -140,6 +168,7 @@
     end
   end
   
   -- Check properties
   if schema.properties and type(value) == "table" then
     for key, prop_schema in pairs(schema.properties) do
@@ -149,6 +178,7 @@
     end
   end
   
+  -- Check array items
   if schema.items and type(value) == "table" then
     for i, item in ipairs(value) do
       local item_errors = validate_value(item, schema.items, path .. "[" .. i .. "]")
@@ -158,6 +188,7 @@
     end
   end
   
+  -- Check enum
   if schema.enum and type(value) ~= "table" then
     local found = false
     for _, enum_val in ipairs(schema.enum) do
@@ -171,6 +202,7 @@
     end
   end
   
+  -- Check oneOf
   if schema.oneOf then
     local valid_count = 0
     for _, sub_schema in ipairs(schema.oneOf) do
@@ -184,6 +216,7 @@
     end
   end
   
+  -- Check allOf
   if schema.allOf then
     for _, sub_schema in ipairs(schema.allOf) do
       local sub_errors = validate_value(value, sub_schema, path)
@@ -193,6 +226,7 @@
     end
   end
   
+  -- Check anyOf
   if schema.anyOf then
     local any_valid = false
     for _, sub_schema in ipairs(schema.anyOf) do
@@ -210,6 +244,7 @@
   return errors
 end
 
+-- Validate a mock response against the OpenAPI schema
 local function validate_mock(path, method, status_code, response_body)
   local schema = get_response_schema(path, method, status_code)
   if not schema then
@@ -217,6 +252,11 @@
   end
   
   local parsed_body
+  
+  -- If response_body is already a table (Lua table), use it directly
+  if type(response_body) == "table" then
+    parsed_body = response_body
+  else
+    -- Try to parse as JSON string
   local ok, result = pcall(function()
     -- Simple JSON parsing for validation
     JSON = {}
@@ -227,6 +267,7