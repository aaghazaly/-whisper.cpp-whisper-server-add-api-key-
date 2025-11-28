# **1. Define an API Key Parameter**

In `server.cpp`, inside the `struct ServerParams` (or wherever server parameters are defined), add a field for the API key:

```cpp
struct ServerParams {
    ...
    std::string api_key; // <-- add this line
    ...
};
```

---

# **2. Add Command Line Option for API Key**

Later in `server.cpp`, command line arguments are parsed (probably using `cxxopts` or a custom parser). Add a new option:

```cpp
("api-key", "Require this API key for protected endpoints (env WHISPER_API_KEY if not provided)", cxxopts::value<std::string>()->default_value(""))
```

Then, after parsing:

```cpp
sparams.api_key = result["api-key"].as<std::string>();

// fallback to environment variable
if (sparams.api_key.empty()) {
    const char* env_key = std::getenv("WHISPER_API_KEY");
    if (env_key) {
        sparams.api_key = env_key;
    }
}
```

---

# **3. Implement `check_api_key` Function**

Add this function near the top of `server.cpp` (after includes):

```cpp
bool check_api_key(const Request &req, Response &res, const ServerParams &sparams) {
    if (sparams.api_key.empty()) {
        return true; // no API key required
    }

    auto auth_header = req.get_header_value("Authorization");
    if (auth_header.empty() || auth_header.find("Bearer ") != 0) {
        res.status = 401;
        res.set_content("{\"error\": \"Unauthorized\"}", "application/json");
        return false;
    }

    std::string token = auth_header.substr(7); // remove "Bearer "
    if (token != sparams.api_key) {
        res.status = 401;
        res.set_content("{\"error\": \"Unauthorized\"}", "application/json");
        return false;
    }

    return true;
}
```

---

# **4. Enforce API Key in Handlers**

Wherever you define the HTTP endpoints (`svr->Post` or `svr->Get`), add the API key check **at the top of the lambda**.

### **4.1 `/inference` Handler**

```cpp
svr->Post(sparams.request_path + sparams.inference_path,
          [&](const Request &req, Response &res) {
    // Check API key
    if (!check_api_key(req, res, sparams)) return;

    std::lock_guard<std::mutex> lock(whisper_mutex);

    // ... rest of your inference code
});
```

---

### **4.2 `/load` Handler**

```cpp
svr->Post(sparams.request_path + "/load",
          [&](const Request &req, Response &res) {
    // Check API key (loading models should be protected)
    if (!check_api_key(req, res, sparams)) return;

    std::lock_guard<std::mutex> lock(whisper_mutex);
    state.store(SERVER_STATE_LOADING_MODEL);

    // ... rest of your load code
});
```

---

### **4.3 `/health` Handler (optional)**

```cpp
svr->Get(sparams.request_path + "/health",
         [&](const Request &req, Response &res) {
    // Optional: protect health endpoint
    if (!check_api_key(req, res, sparams)) return;

    res.set_content("{\"status\":\"ok\"}", "application/json");
});
```

---

# **5. Fix Lambda Parameter Scope Errors**

* Always define the lambda parameters as:

```cpp
[&](const Request &req, Response &res)
```

* **Common mistake:** forgetting `req` in the lambda — you get:
  `error: 'req' was not declared in this scope`

---

# **6. Rebuild the Server**

```bash
cmake --build build -j --config Release
```

---

# **7. Run Server With API Key**

```bash
./build/bin/whisper-server \
  -m /path/to/models/ggml-large-v3.bin \
  --api-key mysecretkey \
  --port 8091
```

* Or use environment variable:

```bash
export WHISPER_API_KEY=mysecretkey
./build/bin/whisper-server -m /path/to/models/ggml-large-v3.bin --port 8091
```

---

# **8. Test With cURL**

✅ **Authorized request:**

```bash
curl -X POST http://127.0.0.1:8091/inference \
-H "Authorization: Bearer mysecretkey" \
-F "file=@jfk.mp3"
```

❌ **Unauthorized request:**

```bash
curl -X POST http://127.0.0.1:8091/inference \
-H "Authorization: Bearer wrongkey" \
-F "file=@jfk.mp3"
```

* Should return **401 Unauthorized**.

---

# ✅ **Summary**

1. Add `api_key` to `ServerParams`.
2. Add CLI option `--api-key`.
3. Implement `check_api_key(req, res, sparams)`.
4. Call `check_api_key` at the **top of every protected handler** (`/inference`, `/load`, optionally `/health`).
5. Ensure lambda signature has `const Request &req, Response &res`.
6. Rebuild and run the server.

