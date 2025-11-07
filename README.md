# Nova-Ride
Next‑gen Ridesharing
// NovaRidePrototype.jsx
// Self-contained front-end prototype for NovaRide — a single-file React component.
// This file contains simulated vehicles, a simple interactive map (canvas),
// a booking form, price/ETA estimates, and lightweight UI state.
// No external libraries required; drop into a Create React App or similar environment.

import React, { useEffect, useRef, useState } from "react";

/*
Design notes:
- The "map" is a simple HTML canvas rendering vehicles as colored circles.
- Vehicles move with small random vectors to simulate availability and movement.
- Booking computes straight-line distance and estimates ETA and price.
- All data is simulated locally; replace simulation logic with real APIs as needed.
*/

const MAP_WIDTH = 800;
const MAP_HEIGHT = 450;
const VEHICLE_COUNT = 12;
const UPDATE_MS = 1000; // movement update interval

// Utility: generate a random number in range [min, max)
function rand(min, max) {
  return Math.random() * (max - min) + min;
}

// Utility: distance on canvas (Euclidean)
function distance(a, b) {
  const dx = a.x - b.x;
  const dy = a.y - b.y;
  return Math.sqrt(dx * dx + dy * dy);
}

// Initial vehicle types and their base pricing / speed characteristics
const VEHICLE_TYPES = {
  Economy: { color: "#2D9CDB", baseFare: 2.0, perKm: 0.8, speed: 40 }, // km/h
  Premium: { color: "#F2994A", baseFare: 4.0, perKm: 1.6, speed: 50 },
  EV: { color: "#27AE60", baseFare: 3.0, perKm: 1.1, speed: 45 },
};

function createRandomVehicles(count) {
  const types = Object.keys(VEHICLE_TYPES);
  const vehicles = [];
  for (let i = 0; i < count; i++) {
    const type = types[i % types.length];
    vehicles.push({
      id: `veh-${i + 1}`,
      type,
      x: rand(40, MAP_WIDTH - 40),
      y: rand(40, MAP_HEIGHT - 40),
      vx: rand(-25, 25) / 10, // px per update
      vy: rand(-25, 25) / 10,
      active: true,
    });
  }
  return vehicles;
}

export default function NovaRidePrototype() {
  const canvasRef = useRef(null);
  const [vehicles, setVehicles] = useState(() => createRandomVehicles(VEHICLE_COUNT));
  const [selectedVehicleType, setSelectedVehicleType] = useState("Economy");
  const [pickup, setPickup] = useState({ x: MAP_WIDTH / 4, y: MAP_HEIGHT / 2 });
  const [dropoff, setDropoff] = useState({ x: (MAP_WIDTH * 3) / 4, y: MAP_HEIGHT / 2 });
  const [matched, setMatched] = useState(null);
  const [estimate, setEstimate] = useState(null);
  const [lastBooked, setLastBooked] = useState(null);

  // Animate vehicles: simple random walk, bouncing off edges
  useEffect(() => {
    const id = setInterval(() => {
      setVehicles((prev) =>
        prev.map((v) => {
          let nx = v.x + v.vx;
          let ny = v.y + v.vy;
          let nvx = v.vx;
          let nvy = v.vy;

          // Bounce on edges with slight damping and small random change
          if (nx < 10 || nx > MAP_WIDTH - 10) {
            nvx = -nvx + rand(-0.5, 0.5);
            nx = Math.max(10, Math.min(MAP_WIDTH - 10, nx));
          }
          if (ny < 10 || ny > MAP_HEIGHT - 10) {
            nvy = -nvy + rand(-0.5, 0.5);
            ny = Math.max(10, Math.min(MAP_HEIGHT - 10, ny));
          }

          // Small random jitter
          nvx += rand(-0.2, 0.2);
          nvy += rand(-0.2, 0.2);

          // Limit speeds
          nvx = Math.max(-3, Math.min(3, nvx));
          nvy = Math.max(-3, Math.min(3, nvy));

          return { ...v, x: nx, y: ny, vx: nvx, vy: nvy };
        })
      );
    }, UPDATE_MS);
    return () => clearInterval(id);
  }, []);

  // Render canvas
  useEffect(() => {
    const canvas = canvasRef.current;
    if (!canvas) return;
    const ctx = canvas.getContext("2d");
    // Clear
    ctx.clearRect(0, 0, MAP_WIDTH, MAP_HEIGHT);

    // Background grid
    ctx.fillStyle = "#f6f8fa";
    ctx.fillRect(0, 0, MAP_WIDTH, MAP_HEIGHT);
    ctx.strokeStyle = "#e6e9ee";
    ctx.lineWidth = 1;
    for (let x = 0; x < MAP_WIDTH; x += 80) {
      ctx.beginPath();
      ctx.moveTo(x, 0);
      ctx.lineTo(x, MAP_HEIGHT);
      ctx.stroke();
    }
    for (let y = 0; y < MAP_HEIGHT; y += 60) {
      ctx.beginPath();
      ctx.moveTo(0, y);
      ctx.lineTo(MAP_WIDTH, y);
      ctx.stroke();
    }

    // Draw pickup & dropoff
    function drawPin(p, label, color) {
      ctx.beginPath();
      ctx.arc(p.x, p.y, 8, 0, Math.PI * 2);
      ctx.fillStyle = color;
      ctx.fill();
      ctx.strokeStyle = "#fff";
      ctx.lineWidth = 2;
      ctx.stroke();
      ctx.fillStyle = "#222";
      ctx.font = "12px sans-serif";
      ctx.fillText(label, p.x + 12, p.y + 4);
    }
    drawPin(pickup, "Pickup", "#FF5A5F");
    drawPin(dropoff, "Dropoff", "#00A699");

    // Draw vehicles
    vehicles.forEach((v) => {
      const cfg = VEHICLE_TYPES[v.type] || VEHICLE_TYPES["Economy"];
      ctx.beginPath();
      ctx.arc(v.x, v.y, 7, 0, Math.PI * 2);
      ctx.fillStyle = cfg.color;
      ctx.fill();
      ctx.strokeStyle = "#fff";
      ctx.lineWidth = 1.5;
      ctx.stroke();
    });

    // If matched vehicle present, highlight path
    if (matched) {
      ctx.beginPath();
      ctx.moveTo(matched.x, matched.y);
      ctx.lineTo(pickup.x, pickup.y);
      ctx.lineTo(dropoff.x, dropoff.y);
      ctx.strokeStyle = "#333";
      ctx.setLineDash([6, 6]);
      ctx.lineWidth = 1.5;
      ctx.stroke();
      ctx.setLineDash([]);
    }
  }, [vehicles, pickup, dropoff, matched]);

  // Find nearest available vehicle of selected type
  function findNearestVehicle(type, from) {
    const candidates = vehicles.filter((v) => v.type === type && v.active);
    if (candidates.length === 0) return null;
    let best = candidates[0];
    let bestDist = distance(from, best);
    for (let i = 1; i < candidates.length; i++) {
      const d = distance(from, candidates[i]);
      if (d < bestDist) {
        best = candidates[i];
        bestDist = d;
      }
    }
    return best;
  }

  // Simulate booking: compute estimate and set matched vehicle
  function requestRide(e) {
    e && e.preventDefault();
    const vehicle = findNearestVehicle(selectedVehicleType, pickup);
    if (!vehicle) {
      setEstimate({ error: "No vehicles available for selected type." });
      setMatched(null);
      return;
    }
    // Convert canvas px distance to "km" using an arbitrary scale for prototype
    const pxToKm = 0.05; // 1 px -> 0.05 km (prototype scale)
    const toPickupKm = distance(vehicle, pickup) * pxToKm;
    const tripKm = distance(pickup, dropoff) * pxToKm;
    const cfg = VEHICLE_TYPES[selectedVehicleType];
    // ETA: time for vehicle to arrive + trip time
    const vehicleSpeedKmh = cfg.speed; // as defined
    const etaMinutes = Math.max(2, Math.round((toPickupKm / vehicleSpeedKmh) * 60));
    const tripMinutes = Math.max(3, Math.round((tripKm / vehicleSpeedKmh) * 60));
    const price = (cfg.baseFare + cfg.perKm * tripKm).toFixed(2);
    const est = {
      vehicleId: vehicle.id,
      etaMinutes,
      tripMinutes,
      tripKm: tripKm.toFixed(2),
      price,
      vehicleType: selectedVehicleType,
      vehicleCoords: { x: vehicle.x, y: vehicle.y },
    };
    setEstimate(est);
    setMatched({ ...vehicle });
  }

  // Confirm booking: mark vehicle as inactive for a short simulated trip
  function confirmBooking() {
    if (!estimate) return;
    const vehicleId = estimate.vehicleId;
    // Mark vehicle inactive and simulate moving from pickup to dropoff over time
    setVehicles((prev) => prev.map((v) => (v.id === vehicleId ? { ...v, active: false } : v)));
    setLastBooked({ ...estimate, bookedAt: new Date().toISOString() });
    // After simulated duration, "complete" trip and respawn vehicle somewhere else
    const totalSimMs = Math.max(5000, (estimate.etaMinutes + estimate.tripMinutes) * 200); // scaled
    setTimeout(() => {
      setVehicles((prev) =>
        prev.map((v) =>
          v.id === vehicleId
            ? {
                ...v,
                x: rand(40, MAP_WIDTH - 40),
                y: rand(40, MAP_HEIGHT - 40),
                vx: rand(-1.5, 1.5),
                vy: rand(-1.5, 1.5),
                active: true,
              }
            : v
        )
      );
      setMatched(null);
      setEstimate(null);
    }, totalSimMs);
  }

  // Quick UI handlers for dragging pickup/dropoff on canvas
  useEffect(() => {
    const canvas = canvasRef.current;
    if (!canvas) return;
    let drag = null;
    function getMousePos(evt) {
      const rect = canvas.getBoundingClientRect();
      return { x: evt.clientX - rect.left, y: evt.clientY - rect.top };
    }
    function onDown(e) {
      const pos = getMousePos(e);
      if (distance(pos, pickup) < 12) drag = "pickup";
      else if (distance(pos, dropoff) < 12) drag = "dropoff";
    }
    function onMove(e) {
      if (!drag) return;
      const pos = getMousePos(e);
      const clamped = {
        x: Math.max(8, Math.min(MAP_WIDTH - 8, pos.x)),
        y: Math.max(8, Math.min(MAP_HEIGHT - 8, pos.y)),
      };
      if (drag === "pickup") setPickup(clamped);
      else setDropoff(clamped);
    }
    function onUp() {
      drag = null;
    }
    canvas.addEventListener("mousedown", onDown);
    window.addEventListener("mousemove", onMove);
    window.addEventListener("mouseup", onUp);
    return () => {
      canvas.removeEventListener("mousedown", onDown);
      window.removeEventListener("mousemove", onMove);
      window.removeEventListener("mouseup", onUp);
    };
  }, [pickup, dropoff]);

  // Minimal styles (inline objects)
  const styles = {
    container: {
      fontFamily: "Inter, Roboto, Arial, sans-serif",
      margin: "18px",
      color: "#111827",
    },
    header: {
      display: "flex",
      justifyContent: "space-between",
      alignItems: "center",
      marginBottom: "12px",
    },
    title: {
      fontSize: "20px",
      fontWeight: 700,
      letterSpacing: "-0.2px",
    },
    layout: {
      display: "grid",
      gridTemplateColumns: "420px 1fr",
      gap: "14px",
      alignItems: "start",
    },
    card: {
      background: "#fff",
      borderRadius: 10,
      padding: 14,
      boxShadow: "0 1px 3px rgba(15,23,42,0.06)",
    },
    smallMuted: { color: "#6B7280", fontSize: 13 },
    formRow: { marginBottom: 10 },
    input: {
      width: "100%",
      padding: "8px 10px",
      borderRadius: 6,
      border: "1px solid #E5E7EB",
      fontSize: 14,
    },
    button: {
      padding: "10px 14px",
      borderRadius: 8,
      border: "none",
      background: "#111827",
      color: "white",
      cursor: "pointer",
      fontWeight: 600,
    },
    secondaryButton: {
      padding: "8px 12px",
      borderRadius: 8,
      border: "1px solid #E5E7EB",
      background: "white",
      cursor: "pointer",
    },
    vehicleList: { marginTop: 8, display: "grid", gap: 8 },
    vehicleRow: { display: "flex", justifyContent: "space-between", alignItems: "center" },
    canvasWrapper: { borderRadius: 10, overflow: "hidden", border: "1px solid #E6E9EE" },
  };

  return (
    <div style={styles.container}>
      <div style={styles.header}>
        <div>
          <div style={styles.title}>NovaRide — Prototype</div>
          <div style={styles.smallMuted}>Simulated front-end demo (interactive)</div>
        </div>
        <div style={styles.smallMuted}>Vehicles: {vehicles.length} • Active: {vehicles.filter(v=>v.active).length}</div>
      </div>

      <div style={styles.layout}>
        <div style={styles.card}>
          <div style={{ display: "flex", justifyContent: "space-between", marginBottom: 10 }}>
            <div>
              <div style={{ fontWeight: 700 }}>Request a Ride</div>
              <div style={styles.smallMuted}>Choose pickup, dropoff, and vehicle type</div>
            </div>
            <div style={{ fontSize: 13, color: "#6B7280" }}>
              Note: Drag pins on the map to adjust positions
            </div>
          </div>

          <form onSubmit={requestRide}>
            <div style={styles.formRow}>
              <label style={{ fontSize: 13, color: "#374151" }}>Vehicle type</label>
              <select
                value={selectedVehicleType}
                onChange={(e) => setSelectedVehicleType(e.target.value)}
                style={{ ...styles.input, marginTop: 6 }}
              >
                {Object.keys(VEHICLE_TYPES).map((t) => (
                  <option key={t} value={t}>
                    {t}
                  </option>
                ))}
              </select>
            </div>

            <div style={styles.formRow}>
              <label style={{ fontSize: 13, color: "#374151" }}>Pickup (x, y)</label>
              <div style={{ display: "flex", gap: 8, marginTop: 6 }}>
                <input
                  style={styles.input}
                  type="number"
                  value={Math.round(pickup.x)}
                  onChange={(e) => setPickup({ ...pickup, x: Math.max(8, Math.min(MAP_WIDTH - 8, Number(e.target.value))) })}
                />
                <input
                  style={styles.input}
                  type="number"
                  value={Math.round(pickup.y)}
                  onChange={(e) => setPickup({ ...pickup, y: Math.max(8, Math.min(MAP_HEIGHT - 8, Number(e.target.value))) })}
                />
              </div>
            </div>

            <div style={styles.formRow}>
              <label style={{ fontSize: 13, color: "#374151" }}>Dropoff (x, y)</label>
              <div style={{ display: "flex", gap: 8, marginTop: 6 }}>
                <input
                  style={styles.input}
                  type="number"
                  value={Math.round(dropoff.x)}
                  onChange={(e) => setDropoff({ ...dropoff, x: Math.max(8, Math.min(MAP_WIDTH - 8, Number(e.target.value))) })}
                />
                <input
                  style={styles.input}
                  type="number"
                  value={Math.round(dropoff.y)}
                  onChange={(e) => setDropoff({ ...dropoff, y: Math.max(8, Math.min(MAP_HEIGHT - 8, Number(e.target.value))) })}
                />
              </div>
            </div>

            <div style={{ display: "flex", gap: 8, marginTop: 8 }}>
              <button type="submit" style={styles.button}>
                Request Ride
              </button>
              <button
                type="button"
                style={styles.secondaryButton}
                onClick={() => {
                  // Reset example scene
                  setVehicles(createRandomVehicles(VEHICLE_COUNT));
                  setMatched(null);
                  setEstimate(null);
                }}
              >
                Reset Scene
              </button>
            </div>
          </form>

          <div style={{ marginTop: 12 }}>
            <div style={{ fontWeight: 700, marginBottom: 6 }}>Estimate</div>
            {estimate ? (
              estimate.error ? (
                <div style={{ color: "#b91c1c" }}>{estimate.error}</div>
              ) : (
                <div style={{ display: "grid", gap: 6 }}>
                  <div style={styles.smallMuted}>Vehicle: {estimate.vehicleId} • {estimate.vehicleType}</div>
                  <div>ETA: {estimate.etaMinutes} min • Trip: {estimate.tripMinutes} min</div>
                  <div>Distance: {estimate.tripKm} km • Price: ${estimate.price}</div>
                  <div style={{ marginTop: 8, display: "flex", gap: 8 }}>
                    <button style={styles.button} onClick={confirmBooking}>
                      Confirm Booking
                    </button>
                    <button
                      style={styles.secondaryButton}
                      onClick={() => {
                        setEstimate(null);
                        setMatched(null);
                      }}
                    >
                      Cancel
                    </button>
                  </div>
                </div>
              )
            ) : (
              <div style={styles.smallMuted}>No estimate yet. Click Request Ride to see a simulated ETA & price.</div>
            )}
          </div>

          <div style={{ marginTop: 14 }}>
            <div style={{ fontWeight: 700, marginBottom: 6 }}>Last booked</div>
            {lastBooked ? (
              <div style={{ fontSize: 13 }}>
                {lastBooked.vehicleId} • ${lastBooked.price} • {new Date(lastBooked.bookedAt).toLocaleTimeString()}
              </div>
            ) : (
              <div style={styles.smallMuted}>No bookings yet.</div>
            )}
          </div>
        </div>

        <div>
          <div style={{ ...styles.card, marginBottom: 12 }}>
            <div style={{ display: "flex", justifyContent: "space-between", marginBottom: 10 }}>
              <div style={{ fontWeight: 700 }}>Map</div>
              <div style={styles.smallMuted}>Click and drag pins to move pickup / dropoff</div>
            </div>
            <div style={styles.canvasWrapper}>
              <canvas ref={canvasRef} width={MAP_WIDTH} height={MAP_HEIGHT} />
            </div>
          </div>

          <div style={styles.card}>
            <div style={{ display: "flex", justifyContent: "space-between", marginBottom: 8 }}>
              <div style={{ fontWeight: 700 }}>Nearby Vehicles</div>
              <div style={styles.smallMuted}>{vehicles.filter(v=>v.type===selectedVehicleType).length} {selectedVehicleType}</div>
            </div>

            <div style={styles.vehicleList}>
              {vehicles
                .slice()
                .sort((a, b) => distance(a, pickup) - distance(b, pickup))
                .slice(0, 8)
                .map((v) => {
                  const cfg = VEHICLE_TYPES[v.type];
                  const distPx = Math.round(distance(v, pickup));
                  return (
                    <div key={v.id} style={styles.vehicleRow}>
                      <div style={{ display: "flex", gap: 10, alignItems: "center" }}>
                        <div style={{ width: 10, height: 10, background: cfg.color, borderRadius: 4 }} />
                        <div>
                          <div style={{ fontWeight: 700 }}>{v.id} <span style={{ color: "#6B7280", fontSize: 12 }}>· {v.type}</span></div>
                          <div style={{ fontSize: 13, color: "#6B7280" }}>{distPx}px away</div>
                        </div>
                      </div>
                      <div style={{ display: "flex", gap: 8 }}>
                        <button
                          style={styles.secondaryButton}
                          onClick={() => {
                            setMatched(v);
                            // Set pickup to be near the vehicle for visualization if user wants to "hail"
                            setPickup({ x: Math.max(8, Math.min(MAP_WIDTH - 8, v.x + 16)), y: Math.max(8, Math.min(MAP_HEIGHT - 8, v.y + 16)) });
                          }}
                        >
                          Hail
                        </button>
                        <button
                          style={styles.button}
                          onClick={() => {
                            setSelectedVehicleType(v.type);
                            setMatched(v);
                            requestRide();
                          }}
                        >
                          Request
                        </button>
                      </div>
                    </div>
                  );
                })}
            </div>
            <div style={{ marginTop: 10, fontSize: 13, color: "#6B7280" }}>
              Tip: Vehicles are simulated. Use "Hail" to bring the pickup near a specific vehicle.
            </div>
          </div>
        </div>
      </div>
    </div>
  );
}
