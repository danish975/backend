{
  "name": "code-runner-backend",
  "version": "1.0.0",
  "main": "server.js",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "body-parser": "^1.20.2",
    "express": "^4.18.2",
    "uuid": "^9.0.0"
  }
}
const express = require("express");
const bodyParser = require("body-parser");
const runRoute = require("./routes/run");

const app = express();
const PORT = 5000;

app.use(bodyParser.json());
app.use("/api/run", runRoute);

app.listen(PORT, () => {
  console.log(`Server running on http://localhost:${PORT}`);
});
const express = require("express");
const router = express.Router();
const { runCode } = require("../runner");

router.post("/", async (req, res) => {
  const { code, language } = req.body;

  try {
    const output = await runCode(code, language);
    res.json({ output });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

module.exports = router;
const fs = require("fs");
const { exec } = require("child_process");
const { v4: uuid } = require("uuid");
const path = require("path");

const runCode = (code, language) => {
  return new Promise((resolve, reject) => {
    const id = uuid();
    const dir = path.join(__dirname, "temp", id);
    fs.mkdirSync(dir, { recursive: true });

    let filename;
    let command;

    switch (language) {
      case "javascript":
        filename = "main.js";
        command = `docker run --rm -v ${dir}:/app node:18-alpine node /app/${filename}`;
        break;
      case "python":
        filename = "main.py";
        command = `docker run --rm -v ${dir}:/app python:3.11-alpine python /app/${filename}`;
        break;
      case "cpp":
        filename = "main.cpp";
        command = `docker run --rm -v ${dir}:/app gcc:latest sh -c "g++ /app/${filename} -o /app/a.out && /app/a.out"`;
        break;
      default:
        return reject(new Error("Unsupported language"));
    }

    const filePath = path.join(dir, filename);
    fs.writeFileSync(filePath, code);

    exec(command, (err, stdout, stderr) => {
      if (err || stderr) {
        return resolve(stderr || err.message);
      }
      return resolve(stdout);
    });
  });
};

module.exports = { runCode };
FROM node:18

WORKDIR /usr/src/app

COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 5000
CMD ["node", "server.js"]




