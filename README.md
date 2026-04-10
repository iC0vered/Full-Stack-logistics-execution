icov-nexus/
│
├── services/
│   ├── gateway-service/
│   ├── auth-service/
│   ├── user-service/
│   ├── load-service/
│   ├── dispatch-service/
│   ├── pricing-service/
│   ├── messaging-service/
│   ├── document-service/
│   ├── cargo-watch-service/
│   ├── shipment-service/
│   ├── integration-service/
│   └── notification-service/
│
├── frontend/
│   └── web-dashboard/
│
├── mobile/
│   └── driver-app/
│
├── infra/
│   ├── docker-compose.yml
│   ├── postgres/
│   ├── redis/
│   └── nginx/
│
├── shared/
│   ├── event-bus/
│   ├── middleware/
│   ├── utils/
│   └── constants/
│
├── database/
│   └── migrations/
│
├── package.json
└── README.md


import { createClient } from "redis";

const redis = createClient();
await redis.connect();

export const EventBus = {
  async publish(event, data) {
    await redis.lPush("events", JSON.stringify({ event, data }));
  },

  async subscribe(handler) {
    while (true) {
      const msg = await redis.brPop("events", 0);
      const eventObj = JSON.parse(msg.element);
      handler(eventObj);
    }
  }
};


import express from "express";
import { v4 as uuid } from "uuid";
import { EventBus } from "../../shared/event-bus/index.js";

const app = express();
app.use(express.json());

const loads = [];

app.post("/loads", async (req, res) => {
  const load = {
    id: uuid(),
    ...req.body,
    status: "POSTED"
  };

  loads.push(load);

  await EventBus.publish("load.created", load);

  res.json(load);
});

app.get("/loads", (req, res) => {
  res.json(loads);
});

app.listen(4001, () => console.log("Load service running"));



import express from "express";

const app = express();
app.use(express.json());

function calculateRate(miles, market) {
  const base = miles * market.avg_per_mile;

  return {
    min: base * 0.9,
    avg: base,
    max: base * 1.25
  };
}

app.post("/rate/calculate", (req, res) => {
  const { miles, market } = req.body;
  res.json(calculateRate(miles, market));
});

app.listen(4002);



import express from "express";

const app = express();
app.use(express.json());

function riskScore(load) {
  let risk = 0;

  if (load.rate < load.market_rate * 0.8) risk += 40;
  if (load.new_broker) risk += 30;
  if (load.reposted) risk += 40;

  return risk;
}

app.post("/analyze", (req, res) => {
  const risk = riskScore(req.body);

  res.json({
    risk,
    level:
      risk > 70 ? "HIGH" :
      risk > 40 ? "MEDIUM" : "LOW"
  });
});

app.listen(4003);



import express from "express";

const app = express();
app.use(express.json());

const chats = {};

app.post("/chat/open", (req, res) => {
  chats[req.body.loadId] = [];
  res.json({ status: "opened" });
});

app.post("/chat/send", (req, res) => {
  const { loadId, message } = req.body;

  chats[loadId].push({
    message,
    time: Date.now()
  });

  res.json({ status: "sent" });
});

app.listen(4004);



import express from "express";

const app = express();
app.use(express.json());

function generateDocs(load) {
  return {
    rateConfirmation: true,
    billOfLading: true,
    dispatchSheet: true,
    esignPacket: true
  };
}

app.post("/generate", (req, res) => {
  res.json(generateDocs(req.body));
});

app.listen(4005);



import express from "express";

const app = express();
app.use(express.json());

function matchDriver(load, drivers) {
  return drivers.sort((a, b) =>
    Math.abs(a.location - load.origin) -
    Math.abs(b.location - load.origin)
  )[0];
}

app.post("/assign", (req, res) => {
  const driver = matchDriver(req.body.load, req.body.drivers);
  res.json(driver);
});

app.listen(4006);



import express from "express";
import proxy from "http-proxy-middleware";

const app = express();

app.use("/loads", proxy({ target: "http://load-service:4001" }));
app.use("/pricing", proxy({ target: "http://pricing-service:4002" }));
app.use("/cargo", proxy({ target: "http://cargo-watch-service:4003" }));
app.use("/chat", proxy({ target: "http://messaging-service:4004" }));
app.use("/docs", proxy({ target: "http://document-service:4005" }));
app.use("/dispatch", proxy({ target: "http://dispatch-service:4006" }));

app.listen(3000);



CREATE TABLE users (
  id UUID PRIMARY KEY,
  role TEXT,
  email TEXT,
  company TEXT
);

CREATE TABLE loads (
  id UUID PRIMARY KEY,
  shipper_id UUID,
  broker_id UUID,
  origin TEXT,
  destination TEXT,
  rate NUMERIC,
  status TEXT
);

CREATE TABLE events (
  id UUID PRIMARY KEY,
  type TEXT,
  payload JSONB,
  created_at TIMESTAMP
);



export default function Dashboard() {
  return (
    <div className="grid grid-cols-3 gap-4">
      <LoadBoard />
      <DispatchMap />
      <RateEngine />
      <CargoWatch />
      <Messaging />
      <Documents />
    </div>
  );
}



import { EventBus } from "../../shared/event-bus/index.js";

export async function activateLoad(load) {
  // 1. Validate
  if (!load.origin || !load.destination) {
    await EventBus.publish("load.rejected", load);
    return { status: "REJECTED" };
  }

  // 2. Risk check hook
  await EventBus.publish("cargo.watch.check", load);

  // 3. Pricing validation hook
  await EventBus.publish("rate.calculate", load);

  // 4. Activate load
  const activated = {
    ...load,
    status: "ACTIVE",
    activatedAt: Date.now()
  };

  // 5. Broadcast to marketplace
  await EventBus.publish("load.activated", activated);

  // 6. Start tender window
  await EventBus.publish("tender.open", activated);

  return activated;
}



import { EventBus } from "../../shared/event-bus/index.js";

export async function onboardUser(user) {
  let status = "PENDING";

  // 1. Basic validation
  if (!user.email || !user.role) {
    await EventBus.publish("user.rejected", user);
    return { status: "REJECTED" };
  }

  // 2. Document check
  const needsDocs = ["carrier", "broker"].includes(user.role);

  // 3. Compliance scoring
  const score =
    user.documents?.length ? 70 : 30;

  status = score > 60 ? "APPROVED" : "REVIEW";

  const onboarded = {
    ...user,
    status,
    complianceScore: score,
    dashboardEnabled: status === "APPROVED"
  };

  await EventBus.publish("user.onboarded", onboarded);

  return onboarded;
}



import express from "express";
import { EventBus } from "../../shared/event-bus/index.js";

const app = express();
app.use(express.json());

const documents = new Map();

/**
 * Upload document
 */
app.post("/upload", async (req, res) => {
  const { entityId, type, url, meta } = req.body;

  const doc = {
    id: Date.now().toString(),
    entityId,
    type,
    url,
    meta,
    verified: false,
    uploadedAt: Date.now()
  };

  documents.set(doc.id, doc);

  await EventBus.publish("doc.uploaded", doc);

  res.json(doc);
});

/**
 * Get documents by entity (load/user)
 */
app.get("/entity/:id", (req, res) => {
  const result = Array.from(documents.values()).filter(
    d => d.entityId === req.params.id
  );

  res.json(result);
});

/**
 * Verify document (compliance engine hook)
 */
app.post("/verify/:id", async (req, res) => {
  const doc = documents.get(req.params.id);

  if (!doc) return res.status(404).send("Not found");

  doc.verified = true;

  await EventBus.publish("doc.verified", doc);

  res.json(doc);
});

app.listen(4010, () => console.log("DocVault running"));



import { createClient } from "redis";

const redis = createClient();
await redis.connect();

export const EventBus = {
  async emit(event, data) {
    await redis.lPush("nexus_events", JSON.stringify({ event, data }));
  },

  async listen(handler) {
    while (true) {
      const msg = await redis.brPop("nexus_events", 0);
      const payload = JSON.parse(msg.element);
      handler(payload);
    }
  }
};



load.created
→ auto-onboarding.check (if new user)
→ pricing.calculate
→ cargo.watch.scan



load.created
→ load.activation.service
→ docvault.requireDocs
→ tender.open



tender.open
→ docvault.verifyRequiredDocs

IF docs missing:
    → BLOCK tender
    → BLOCK dispatch
    → FLAG compliance



tender.opened
→ carriers notified
→ bidding begins
→ offers submitted



tender.closed
→ best offer selected
→ dispatch engine assigns carrier
→ route optimization triggered



dispatch.confirmed
→ DocVault generates:
   - rate confirmation
   - BOL
   - dispatch sheet
   - e-sign packet

→ sent to messaging system



load.assigned
→ chat opens automatically
→ docs injected into chat
→ broker + carrier connected



gps.update
→ cargo.watch.monitor
→ compliance tracking active
→ alerts triggered if deviation



pod.uploaded
→ DocVault verifies POD
→ payment released
→ load closed
→ audit log finalized



import { EventBus } from "../shared/event-bus/index.js";

export function startNexusEngine() {
  EventBus.listen(async ({ event, data }) => {

    switch (event) {

      case "load.created":
        console.log("Load created");
        await EventBus.emit("pricing.calculate", data);
        await EventBus.emit("cargo.watch.scan", data);
        await EventBus.emit("load.activation", data);
        break;

      case "load.activation":
        console.log("Activating load");
        await EventBus.emit("tender.open", data);
        await EventBus.emit("docvault.require", data);
        break;

      case "tender.closed":
        await EventBus.emit("dispatch.assign", data);
        break;

      case "dispatch.confirmed":
        await EventBus.emit("docvault.generate", data);
        await EventBus.emit("messaging.open", data);
        break;

      case "pod.uploaded":
        await EventBus.emit("payment.release", data);
        await EventBus.emit("audit.finalize", data);
        break;
    }
  });
}



Events DocVault listens to:

- load.created
- load.activation
- tender.open
- dispatch.confirmed
- pod.uploaded



Listens to:
- load.created
- gps.update
- route.change

Triggers:
- cargo.risk.alert
- compliance.flag



Triggered by:
- load.created
- lane.matching
- market.update

Feeds into:
- tender.minimum_bid
- fraud detection



Triggered by:
- tender.closed

Outputs:
- carrier assigned
- route optimized
- load locked



icov-nexus/
│
├── services/
│   ├── gateway/
│   ├── load-service/
│   ├── dispatch-service/
│   ├── pricing-service/
│   ├── docvault-service/
│   ├── tender-service/
│   ├── cargo-watch-service/
│   ├── onboarding-service/
│   └── worker-engine/
│
├── frontend/
│   └── react-dashboard/
│
├── infra/
│   ├── docker-compose.yml
│   ├── postgres/
│   └── redis/
│
├── shared/
│   ├── event-bus.js
│   ├── db.js
│
└── .env



version: "3.9"

services:

  postgres:
    image: postgres:15
    environment:
      POSTGRES_USER: nexus
      POSTGRES_PASSWORD: nexus
      POSTGRES_DB: nexus
    ports:
      - "5432:5432"

  redis:
    image: redis:7
    ports:
      - "6379:6379"

  gateway:
    build: ./services/gateway
    ports:
      - "3000:3000"
    depends_on:
      - redis
      - postgres

  load-service:
    build: ./services/load-service
    ports:
      - "4001:4001"
    depends_on:
      - redis
      - postgres

  dispatch-service:
    build: ./services/dispatch-service
    ports:
      - "4002:4002"

  pricing-service:
    build: ./services/pricing-service
    ports:
      - "4003:4003"

  docvault-service:
    build: ./services/docvault-service
    ports:
      - "4010:4010"

  tender-service:
    build: ./services/tender-service
    ports:
      - "4011:4011"

  cargo-watch-service:
    build: ./services/cargo-watch-service
    ports:
      - "4012:4012"

  worker-engine:
    build: ./services/worker-engine
    depends_on:
      - redis



import { createClient } from "redis";

const redis = createClient({ url: "redis://redis:6379" });
await redis.connect();

export const EventBus = {
  async emit(event, data) {
    await redis.lPush("events", JSON.stringify({ event, data }));
  },

  async listen(handler) {
    while (true) {
      const msg = await redis.brPop("events", 0);
      handler(JSON.parse(msg.element));
    }
  }
};



import { EventBus } from "../shared/event-bus.js";

console.log("Worker Engine Started");

EventBus.listen(async ({ event, data }) => {

  switch (event) {

    case "load.created":
      console.log("Processing load:", data.id);
      await EventBus.emit("pricing.calculate", data);
      await EventBus.emit("cargo.watch.scan", data);
      await EventBus.emit("tender.open", data);
      break;

    case "tender.closed":
      await EventBus.emit("dispatch.assign", data);
      break;

    case "dispatch.confirmed":
      await EventBus.emit("docvault.generate", data);
      break;

    case "pod.uploaded":
      await EventBus.emit("payment.release", data);
      break;
  }
});



import express from "express";
import { v4 as uuid } from "uuid";
import { EventBus } from "../../shared/event-bus.js";

const app = express();
app.use(express.json());

const loads = [];

app.post("/loads", async (req, res) => {
  const load = {
    id: uuid(),
    ...req.body,
    status: "CREATED"
  };

  loads.push(load);

  await EventBus.emit("load.created", load);

  res.json(load);
});

app.get("/loads", (req, res) => res.json(loads));

app.listen(4001);


import express from "express";

const app = express();
app.use(express.json());

function rateFloor(miles, market) {
  const base = miles * market.avg;

  return {
    min: base * 0.9,
    avg: base,
    max: base * 1.25
  };
}

app.post("/calculate", (req, res) => {
  res.json(rateFloor(req.body.miles, req.body.market));
});

app.listen(4003);




import express from "express";

const app = express();
app.use(express.json());

const docs = new Map();

app.post("/generate", (req, res) => {
  const output = {
    rateConfirmation: true,
    billOfLading: true,
    eSignPacket: true
  };

  docs.set(req.body.id, output);

  res.json(output);
});

app.listen(4010);



import express from "express";

const app = express();
app.use(express.json());

app.post("/assign", (req, res) => {
  const driver = req.body.drivers[0]; // simplified matching

  res.json({
    assigned: driver,
    status: "DISPATCHED"
  });
});

app.listen(4002);



import express from "express";

const app = express();
app.use(express.json());

function risk(load) {
  return load.rate < load.market * 0.8 ? 80 : 20;
}

app.post("/scan", (req, res) => {
  res.json({
    risk: risk(req.body),
    status: "OK"
  });
});

app.listen(4012);



import express from "express";
import proxy from "http-proxy-middleware";

const app = express();

app.use("/loads", proxy({ target: "http://load-service:4001" }));
app.use("/dispatch", proxy({ target: "http://dispatch-service:4002" }));
app.use("/pricing", proxy({ target: "http://pricing-service:4003" }));
app.use("/docs", proxy({ target: "http://docvault-service:4010" }));
app.use("/cargo", proxy({ target: "http://cargo-watch-service:4012" }));

app.listen(3000);



export default function Dashboard() {
  return (
    <div className="grid grid-cols-3 gap-4">
      <LoadBoard />
      <TenderMarket />
      <DispatchCenter />
      <CargoWatch />
      <DocVault />
      <PricingEngine />
    </div>
  );
}



load.created
→ pricing.calculate
→ cargo.watch.scan
→ tender.open
→ bids come in
→ tender.closed
→ dispatch.assign
→ docvault.generate
→ load active



pod.uploaded
→ payment released
→ audit logged
→ load closed



import express from "express";
import jwt from "jsonwebtoken";
import bcrypt from "bcryptjs";

const app = express();
app.use(express.json());

const USERS = new Map();

const SECRET = process.env.JWT_SECRET || "supersecret";

/**
 * REGISTER
 */
app.post("/register", async (req, res) => {
  const { email, password, role } = req.body;

  const hash = await bcrypt.hash(password, 10);

  USERS.set(email, {
    email,
    password: hash,
    role,
    verified: false
  });

  res.json({ status: "registered" });
});

/**
 * LOGIN
 */
app.post("/login", async (req, res) => {
  const user = USERS.get(req.body.email);
  if (!user) return res.status(401).send("Invalid");

  const match = await bcrypt.compare(req.body.password, user.password);
  if (!match) return res.status(401).send("Invalid");

  const token = jwt.sign(
    { email: user.email, role: user.role },
    SECRET,
    { expiresIn: "12h" }
  );

  res.json({ token });
});

/**
 * RBAC Middleware
 */
export function auth(roleList = []) {
  return (req, res, next) => {
    const token = req.headers.authorization?.split(" ")[1];

    try {
      const decoded = jwt.verify(token, SECRET);

      if (roleList.length && !roleList.includes(decoded.role)) {
        return res.status(403).send("Forbidden");
      }

      req.user = decoded;
      next();
    } catch {
      return res.status(401).send("Unauthorized");
    }
  };
}

app.listen(5000);



generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id        String  @id @default(uuid())
  email     String  @unique
  role      String
  verified  Boolean @default(false)
  createdAt DateTime @default(now())
}

model Load {
  id          String  @id @default(uuid())
  origin      String
  destination String
  status      String
  rate        Float
  createdAt   DateTime @default(now())
}

model Tender {
  id      String @id @default(uuid())
  loadId  String
  status  String
}



import { Queue, Worker } from "bullmq";
import IORedis from "ioredis";

const connection = new IORedis({ maxRetriesPerRequest: null });

export const loadQueue = new Queue("load-events", { connection });

export const worker = new Worker(
  "load-events",
  async job => {
    const { event, data } = job.data;

    switch (event) {
      case "load.created":
        console.log("Processing load safely:", data.id);
        break;

      case "tender.open":
        console.log("Opening tender:", data.id);
        break;
    }
  },
  { connection }
);



import { WebSocketServer } from "ws";

const wss = new WebSocketServer({ port: 7071 });

const clients = new Set();

wss.on("connection", socket => {
  clients.add(socket);

  socket.on("message", msg => {
    console.log("market event:", msg);
  });

  socket.on("close", () => clients.delete(socket));
});

export function broadcast(event) {
  for (const c of clients) {
    c.send(JSON.stringify(event));
  }
}



import express from "express";
import { PrismaClient } from "@prisma/client";
import { loadQueue } from "../queues/loadQueue.js";

const app = express();
const prisma = new PrismaClient();

app.use(express.json());

app.post("/loads", async (req, res) => {
  const load = await prisma.load.create({
    data: {
      origin: req.body.origin,
      destination: req.body.destination,
      status: "CREATED",
      rate: req.body.rate
    }
  });

  await loadQueue.add("load.created", {
    event: "load.created",
    data: load
  });

  res.json(load);
});

app.listen(4001);



import { PrismaClient } from "@prisma/client";

const prisma = new PrismaClient();

export async function calculateRate(load) {
  const marketAvg = load.rate || 2.5;

  const min = marketAvg * 0.9;
  const max = marketAvg * 1.25;

  await prisma.load.update({
    where: { id: load.id },
    data: { rate: marketAvg }
  });

  return { min, max, avg: marketAvg };
}



import express from "express";

const app = express();
app.use(express.json());

const DOCS = new Map();

app.post("/upload", (req, res) => {
  const doc = {
    id: Date.now().toString(),
    type: req.body.type,
    entityId: req.body.entityId,
    verified: false
  };

  DOCS.set(doc.id, doc);

  res.json(doc);
});

app.post("/verify/:id", (req, res) => {
  const doc = DOCS.get(req.params.id);
  doc.verified = true;
  res.json(doc);
});

app.listen(4010);



export function assignCarrier(load, carriers) {
  const ranked = carriers.sort(
    (a, b) => a.score - b.score
  );

  return {
    loadId: load.id,
    assignedCarrier: ranked[0],
    status: "ASSIGNED"
  };
}



import express from "express";
import proxy from "express-http-proxy";

const app = express();

app.use("/auth", proxy("http://auth-service:5000"));
app.use("/loads", proxy("http://load-service:400
