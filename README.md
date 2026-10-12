AtmoSync: Micro-Climate Arbitrage Analytics

Detecting when a shipping container's internal conditions are spoiling its cargo, and working out whether rerouting to a closer market would save money.

Built with: Python, SQL (SQLite), pandas, matplotlib, Jupyter

This was a group project completed during a Data Analyst internship at Infotact Solutions Pvt. Ltd. AI tools were used as a coding and brainstorming assistant during development.

All data in this repository is simulated. The results demonstrate the method, not real market findings.



The problem

Traditional supply chain analytics rely on standard transit times and regional weather forecasts. They miss what actually spoils fresh produce: the micro-climate inside one container. A cooling unit that drifts a few degrees, a door seal that lets humidity spike, or rough handling that bruises the fruit all change how fast cargo loses quality, long before it reaches the market.

The idea: Spoilage Arbitrage

For every container, AtmoSync forecasts the cargo's quality on arrival, converts that into expected revenue, and compares two options:

Stay on course to the primary market (higher price, longer trip, more spoilage)
Reroute to a nearer secondary market (lower price, fresher cargo, extra diversion cost)

The difference in net revenue is the spoilage arbitrage. A reroute is recommended only when it clears a minimum gain.

What the notebook does

Everything lives in one notebook, AtmoSync_Analysis.ipynb, and runs top to bottom with no external services.

Stage	Implementation
Sensor simulation	Six containers send temperature, humidity and vibration every 15 minutes. Three have injected faults, three are healthy controls. Dropped readings, firmware glitches, duplicates and out-of-order events are included on purpose
Streaming ingestion	An in-memory event log with topics, partitions, offsets and consumer commits (at-least-once delivery), processed in 3-hour micro-batches
Data warehouse	A SQLite file (atmosync.db) holding raw sensor events, commodity prices and container data
Transformation	Five layered SQL models: staging, intermediate, facts and two marts
Alerting	A rule engine that runs after every micro-batch and prints a live alert feed, including when alerts are withdrawn
Dashboard	A matplotlib dashboard built from the mart tables: KPIs, quality curves, arrival forecasts, arbitrage, temperature trace and price history
Validation	Detection lag for each injected fault, and a false-alarm check on the healthy containers
Injected incidents
Container	Cargo	Incident	Starts
A	Avocados	Cooling unit drifts upward, up to +10 C	Hour 72
D	Avocados	Door seal fails, humidity jumps to about 98.5%	Hour 30
C	Mangoes	Rough seas, vibration spikes for 36 hours	Hour 96
B, E, F	Bananas, bananas, mangoes	None (control group)	n/a
SQL models
Layer	Model	Purpose
Staging	stg_sensor_events	Deduplicate and drop nulls and impossible readings
Intermediate	int_degradation	Join spoilage parameters and compute each reading's loss rate
Facts	fct_quality	Running quality score per container
Marts	mart_container_health	Latest snapshot, recent trends and health status
Marts	mart_spoilage_arbitrage	Forecast quality on arrival, revenue per market, reroute recommendation

Loss is integrated over the actual time between valid readings (capped at 2 hours), so a dropped reading does not distort the quality score. Price lookups stop at the latest sensor timestamp, so early micro-batches never see future prices.

Spoilage model
loss_rate (points/hour) = base_rate x Q10 ^ ((T - T_optimal) / 10)       temperature effect
                          x (1 + k_h x humidity_outside_band_%)           humidity effect
                          + k_v x max(0, vibration - threshold)           bruising from rough handling

Quality starts at 100 on departure. Q10 is the rule that spoilage speeds up by a fixed factor for every 10 C above the optimal storage temperature. A quality-to-price curve then converts forecast arrival quality into a price multiplier, so falling below the quality threshold costs real money.



Alert rules
Alert	Fires when
TEMP_DRIFT	3-hour mean temperature is more than 2 C from the setpoint
HUMIDITY_SPIKE	3-hour mean humidity is more than 3 points outside the optimal band
VIBRATION	3-hour mean vibration exceeds 1.5 g
SPOILAGE_RATE	The current loss rate is at least 1.5x the healthy baseline
REROUTE_RECOMMENDED	Net revenue at the secondary market beats the primary by more than the minimum arbitrage
Results (simulated data)
Metric	Result
Micro-batches processed	48
Raw events landed	3,498 (3,456 expected)
Duplicates removed / null readings / firmware glitches	42 / 57 / 22
Clean readings used downstream	3,377 (97.7% of expected)
Cooling drift (Container A) detected after	14.8 hours
Humidity spike (Container D) detected after	5.8 hours
Rough handling (Container C) detected after	11.8 hours
False alarms on healthy containers	0
Reroutes recommended at day 6	A (about $14,968) and D (about $10,464)
Total spoilage arbitrage available	about $25,432

Container C raised a reroute alert during its vibration spike and then withdrew it once the disturbance ended. This shows a real limitation of the forecast, covered below.

Figures are fixed by the random seed (SEED = 7), so a fresh run should reproduce them. Detection times include the 3-hour micro-batch interval and the 3-hour averaging window.



Getting started

Requirements: Python 3.9 or newer. SQLite 3.25 or newer is needed for window functions, and current Python installers already include it.

bash
pip install pandas numpy matplotlib jupyter
jupyter notebook

Open AtmoSync_Analysis.ipynb and choose Kernel, Restart & Run All. The notebook creates a fresh atmosync.db next to itself each time it runs, so the file can be deleted safely.

Configuration

These constants are set in the first code cell and flow through the SQL models:

Constant	Default	Meaning
SEED	7	Random seed for reproducible results
STREAM_HOURS	144	Hours of the voyage to replay before "going live"
BATCH_HOURS	3	Micro-batch size
QUALITY_THRESHOLD	70	Quality below which cargo loses its premium grade
MIN_ARBITRAGE_USD	2000	Minimum net gain before a reroute is recommended

Spoilage parameters per commodity, container routes and the quality-to-price curve are defined in section 2 of the notebook.

Repository structure
.
├── AtmoSync_Analysis.ipynb   # the full pipeline, with saved outputs
├── README.md
└── images/                   # charts used in this README
Limitations
Simulated data. Sensors, prices, routes and cargo are generated. The spoilage parameters are plausible but not calibrated to a published shelf-life study.
Persistence forecast. The forecast assumes the current loss rate continues until arrival. It overreacts to short disturbances (as Container C showed) and underestimates a drift that is still growing.
One-sided temperature model. Chilling injury (too cold) and ripening stage are not modelled.
Static prices and routes. Prices use a trailing 7-day average and arrival times are fixed. Real decisions would need live quotes, port congestion and diversion feasibility.
Full-refresh models. Every micro-batch rebuilds every table. At larger volumes the models would need to be incremental.
Static dashboard. It is a snapshot at the end of the replay, not a live, filterable screen.
Possible improvements
Replace the in-memory log with a real message broker and SQLite with a cloud warehouse. The SQL models carry over with small dialect changes.
Forecast the sensor trend instead of assuming the current rate persists.
Calibrate the spoilage parameters and price curve with post-harvest research and real sale data.
Add a what-if panel that varies the minimum arbitrage and diversion cost.
Build an interactive version of the dashboard.
Acknowledgements

Thanks to Infotact Solutions Pvt. Ltd. for the internship opportunity and project brief, for guidance, and to my teammates.
