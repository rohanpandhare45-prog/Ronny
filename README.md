const express = require('express');
const path = require('path');
const sqlite3 = require('sqlite3').verbose();
const bodyParser = require('body-parser');

const DB_PATH = path.join(__dirname, 'parking.db');
const PORT = process.env.PORT || 3000;
const RATE_PER_HOUR = 20; // currency units per hour

const app = express();
app.use(bodyParser.json());
app.use(express.static(path.join(__dirname, 'public')));

// initialize DB
const db = new sqlite3.Database(DB_PATH, (err) => {
  if (err) {
    console.error('Failed to open DB', err);
    process.exit(1);
  }
});

db.serialize(() => {
  db.run(`CREATE TABLE IF NOT EXISTS slots (
    id INTEGER PRIMARY KEY,
    occupied INTEGER DEFAULT 0,
    vehicle_number TEXT,
    entry_time INTEGER,
    exit_time INTEGER,
    fee REAL DEFAULT 0
  )`);
  db.run(`CREATE TABLE IF NOT EXISTS logs (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    vehicle_number TEXT,
    slot_id INTEGER,
    entry_time INTEGER,
    exit_time INTEGER,
    fee REAL
  )`);
  // create 20 slots if not exist
  db.get("SELECT COUNT(*) as cnt FROM slots", (err, row) => {
    if (err) throw err;
    if (row.cnt === 0) {
      const stmt = db.prepare("INSERT INTO slots (id) VALUES (?)");
      for (let i=1;i<=20;i++) stmt.run(i);
      stmt.finalize();
      console.log('Created 20 parking slots');
    }
  });
});

// helper
function calcFee(entryTs, exitTs) {
  const ms = exitTs - entryTs;
  const hours = Math.ceil(ms / (1000*60*60)); // charge by full hour
  return hours * RATE_PER_HOUR;
}

// APIs

// get slots
app.get('/api/slots', (req, res) => {
  db.all("SELECT * FROM slots ORDER BY id", (err, rows) => {
    if (err) return res.status(500).json({error: err.message});
    res.json(rows);
  });
});

// vehicle entry
app.post('/api/entry', (req, res) => {
  const { vehicle_number } = req.body;
  if (!vehicle_number) return res.status(400).json({error: 'vehicle_number required'});
  // find first free slot
  db.get("SELECT * FROM slots WHERE occupied=0 ORDER BY id LIMIT 1", (err, slot) => {
    if (err) return res.status(500).json({error: err.message});
    if (!slot) return res.status(400).json({error: 'No free slots'});
    const ts = Date.now();
    db.run("UPDATE slots SET occupied=1, vehicle_number=?, entry_time=?, exit_time=NULL, fee=0 WHERE id=?", [vehicle_number, ts, slot.id], function(err){
      if (err) return res.status(500).json({error: err.message});
      db.run("INSERT INTO logs (vehicle_number, slot_id, entry_time) VALUES (?,?,?)", [vehicle_number, slot.id, ts]);
      res.json({message: 'Entry recorded', slot_id: slot.id, entry_time: ts});
    });
  });
});

// vehicle exit
app.post('/api/exit', (req, res) => {
  const { vehicle_number } = req.body;
  if (!vehicle_number) return res.status(400).json({error: 'vehicle_number required'});
  db.get("SELECT * FROM slots WHERE occupied=1 AND vehicle_number=?", [vehicle_number], (err, slot) => {
    if (err) return res.status(500).json({error: err.message});
    if (!slot) return res.status(400).json({error: 'Vehicle not found in any occupied slot'});
    const exitTs = Date.now();
    const fee = calcFee(slot.entry_time, exitTs);
    db.run("UPDATE slots SET occupied=0, vehicle_number=NULL, entry_time=NULL, exit_time=?, fee=? WHERE id=?", [exitTs, fee, slot.id], function(err){
      if (err) return res.status(500).json({error: err.message});
      // update latest log for this vehicle & slot without exit_time
      db.run("UPDATE logs SET exit_time=?, fee=? WHERE slot_id=? AND vehicle_number=? AND exit_time IS NULL", [exitTs, fee, slot.id, vehicle_number]);
      res.json({message: 'Exit recorded', slot_id: slot.id, exit_time: exitTs, fee});
    });
  });
});

// get logs
app.get('/api/logs', (req, res) => {
  db.all("SELECT * FROM logs ORDER BY id DESC LIMIT 200", (err, rows) => {
    if (err) return res.status(500).json({error: err.message});
    res.json(rows);
  });
});

// get summary
app.get('/api/summary', (req, res) => {
  db.get("SELECT COUNT(*) as total FROM slots", (err, totalRow) => {
    if (err) return res.status(500).json({error: err.message});
    db.get("SELECT COUNT(*) as occupied FROM slots WHERE occupied=1", (err2, occRow) => {
      if (err2) return res.status(500).json({error: err2.message});
      res.json({total: totalRow.total, occupied: occRow.occupied, free: totalRow.total - occRow.occupied});
    });
  });
});

app.get('*', (req, res) => {
  res.sendFile(path.join(__dirname, 'public', 'index.html'));
});

app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
  console.log(`RATE_PER_HOUR = ${RATE_PER_HOUR}`);
});
