GLOBAL POPULATION ANALYTICS SYSTEM (GPAS)
=======================================

TECH STACK
----------
Frontend: React (Vite)
Backend: Node.js + Express
Database: PostgreSQL
Data Source: CSV (World Population Dataset)
Security: .env (Environment Variables)

=======================================
PREREQUISITES (INSTALL FIRST)
=======================================

1) Install Node.js (LTS)
https://nodejs.org

Verify:
node -v
npm -v

2) Install PostgreSQL
https://www.postgresql.org/download/

During install:
- Username: postgres
- Password: remember it
- Port: 5432

Verify:
psql --version

=======================================
PROJECT SETUP
=======================================

mkdir gpas
cd gpas
mkdir backend frontend

=======================================
BACKEND SETUP (EXPRESS + POSTGRESQL)
=======================================

cd backend
npm init -y
npm install express cors pg dotenv csv-parser

---------------------------------------
BACKEND FOLDER STRUCTURE
---------------------------------------

backend/
config/
  db.js
routes/
  statsRoutes.js
scripts/
  importPopulationCsv.js
data/
  world_population.csv
index.js
.env
package.json

---------------------------------------
ENVIRONMENT VARIABLES
---------------------------------------

Create file: backend/.env

PORT=3000
DB_HOST=localhost
DB_PORT=5432
DB_NAME=world_population_db
DB_USER=postgres
DB_PASSWORD=your_password_here

---------------------------------------
POSTGRESQL DATABASE & TABLES
---------------------------------------

CREATE DATABASE world_population_db;

CREATE TABLE regions (
  id SERIAL PRIMARY KEY,
  name VARCHAR(50) UNIQUE NOT NULL
);

CREATE TABLE continents (
  id SERIAL PRIMARY KEY,
  name VARCHAR(50) UNIQUE NOT NULL,
  region_id INT REFERENCES regions(id)
);

CREATE TABLE countries (
  id SERIAL PRIMARY KEY,
  name VARCHAR(100) UNIQUE NOT NULL,
  iso_code CHAR(3),
  capital VARCHAR(100),
  continent_id INT REFERENCES continents(id),
  area_km2 BIGINT,
  density NUMERIC,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE population (
  id SERIAL PRIMARY KEY,
  country_id INT REFERENCES countries(id) ON DELETE CASCADE,
  year INT NOT NULL,
  population BIGINT NOT NULL,
  growth_rate NUMERIC,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  UNIQUE (country_id, year)
);

CREATE TABLE population_stats (
  id SERIAL PRIMARY KEY,
  country_id INT REFERENCES countries(id),
  stat_type VARCHAR(20),
  year_from INT,
  year_to INT,
  value BIGINT,
  generated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

---------------------------------------
DATABASE CONNECTION (backend/config/db.js)
---------------------------------------

const { Pool } = require("pg");

const pool = new Pool({
  host: process.env.DB_HOST,
  port: process.env.DB_PORT,
  database: process.env.DB_NAME,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD
});

module.exports = pool;

---------------------------------------
CSV IMPORT SCRIPT
(backend/scripts/importPopulationCsv.js)
---------------------------------------

require("dotenv").config();
const fs = require("fs");
const csv = require("csv-parser");
const pool = require("../config/db");

const YEARS = [1970,1980,1990,2000,2010,2015,2020,2022];

async function getOrCreate(table, column, value, extra = {}) {
  const found = await pool.query(
    `SELECT id FROM ${table} WHERE ${column}=$1`,
    [value]
  );

  if (found.rows.length) return found.rows[0].id;

  const keys = Object.keys(extra);
  const values = Object.values(extra);
  const cols = [column, ...keys].join(",");
  const params = [value, ...values];
  const placeholders = params.map((_, i) => `$${i+1}`).join(",");

  const result = await pool.query(
    `INSERT INTO ${table} (${cols})
     VALUES (${placeholders})
     RETURNING id`,
    params
  );

  return result.rows[0].id;
}

fs.createReadStream("./data/world_population.csv")
  .pipe(csv())
  .on("data", async row => {
    const regionId = await getOrCreate("regions", "name", row["Region"]);
    const continentId = await getOrCreate(
      "continents", "name", row["Continent"], { region_id: regionId }
    );
    const countryId = await getOrCreate(
      "countries", "name", row["Country/Territory"], {
        iso_code: row["CCA3"],
        capital: row["Capital"],
        continent_id: continentId,
        area_km2: row["Area (km²)"],
        density: row["Density (per km²)"]
      }
    );

    let prev = null;
    for (const year of YEARS) {
      const pop = Number(row[`${year} Population`]);
      if (!pop) continue;

      const growth = prev ? ((pop - prev) / prev) * 100 : null;

      await pool.query(
        `INSERT INTO population (country_id, year, population, growth_rate)
         VALUES ($1,$2,$3,$4)
         ON CONFLICT DO NOTHING`,
        [countryId, year, pop, growth]
      );

      prev = pop;
    }
  })
  .on("end", () => {
    console.log("CSV IMPORT COMPLETED");
    pool.end();
  });

---------------------------------------
STATS API (backend/routes/statsRoutes.js)
---------------------------------------

const express = require("express");
const pool = require("../config/db");
const router = express.Router();

router.get("/most-populated", async (req, res) => {
  const { from, to } = req.query;

  const result = await pool.query(
    `SELECT c.name, MAX(p.population) AS max_population
     FROM population p
     JOIN countries c ON c.id = p.country_id
     WHERE p.year BETWEEN $1 AND $2
     GROUP BY c.name
     ORDER BY max_population DESC
     LIMIT 1`,
    [from, to]
  );

  res.json(result.rows[0]);
});

module.exports = router;

---------------------------------------
BACKEND ENTRY FILE (backend/index.js)
---------------------------------------

require("dotenv").config();
const express = require("express");
const cors = require("cors");
const statsRoutes = require("./routes/statsRoutes");

const app = express();
app.use(cors());
app.use(express.json());
app.use("/api/stats", statsRoutes);

app.listen(process.env.PORT, () => {
  console.log("Backend running on port", process.env.PORT);
});

---------------------------------------
RUN BACKEND
---------------------------------------

node scripts/importPopulationCsv.js
node index.js

=======================================
FRONTEND SETUP (REACT)
=======================================

cd ../frontend
npm create vite@latest .
npm install
npm install axios chart.js react-chartjs-2
npm run dev

---------------------------------------
FRONTEND API SERVICE
(frontend/src/services/api.js)
---------------------------------------

import axios from "axios";

export default axios.create({
  baseURL: "http://localhost:3000/api"
});

---------------------------------------
DASHBOARD COMPONENT
(frontend/src/components/Dashboard.jsx)
---------------------------------------

import StatsWidget from "./StatsWidget";

const Dashboard = () => (
  <div style={{ padding: 40 }}>
    <h1>Global Population Analytics System</h1>
    <StatsWidget />
  </div>
);

export default Dashboard;

---------------------------------------
STATS WIDGET
(frontend/src/components/StatsWidget.jsx)
---------------------------------------

import { useEffect, useState } from "react";
import api from "../services/api";

const StatsWidget = () => {
  const [data, setData] = useState(null);

  useEffect(() => {
    api.get("/stats/most-populated?from=2020&to=2022")
      .then(res => setData(res.data));
  }, []);

  return (
    <div>
      <h3>Most Populated Country (2020–2022)</h3>
      {data && <p>{data.name} — {data.max_population}</p>}
    </div>
  );
};

export default StatsWidget;

---------------------------------------
APP ENTRY
(frontend/src/App.jsx)
---------------------------------------

import Dashboard from "./components/Dashboard";
export default () => <Dashboard />;

=======================================
OPEN APPLICATION
=======================================

Frontend:
http://localhost:5173

Backend API:
http://localhost:3000/api/stats/most-populated?from=2020&to=2022

=======================================
END OF COMPLETE PROJECT
=======================================
