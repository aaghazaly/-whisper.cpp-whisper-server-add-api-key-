# whisper.cpp/examples/server
1- replace your server.cpp with the server.cpp file in this repositroy 
2- run  : cmake --build build -j --config Release
3- start server with api-key : ./build/bin/whisper-server  -m /path/whisper-cpp/whisper.cpp/models/ggml-large-v3.bin --api-key mysecretkey   --port 8091

# Or via env var:
WHISPER_API_KEY=mysecretkey ./build/bin/whisper-server


4- send request using curl to test : 
# in samples folder 
curl -v -X POST http://127.0.0.1:8091/inference \
  -H "Authorization: Bearer mysecretkey" \
  -F "file=@jfk.mp3"

5- for help you can run ./build/bin/whisper-server -h
