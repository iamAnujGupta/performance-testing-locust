# Performance Testing with Locust

![Python](https://img.shields.io/badge/Python-3.9+-3776AB?style=flat-square&logo=python&logoColor=white)
![Locust](https://img.shields.io/badge/Locust-2.x-5C5543?style=flat-square)
![Grafana](https://img.shields.io/badge/Grafana-Dashboards-F46800?style=flat-square&logo=grafana&logoColor=white)
![Prometheus](https://img.shields.io/badge/Prometheus-Metrics-E6522C?style=flat-square&logo=prometheus&logoColor=white)
![S3](https://img.shields.io/badge/AWS_S3-Compatible-232F3E?style=flat-square&logo=amazonaws&logoColor=white)

S3 object storage **performance and load testing framework** using **Locust**. Covers throughput benchmarking, latency profiling, and bottleneck analysis for S3-compatible APIs (DataCore Swarm, MinIO, AWS S3). Real-time Grafana/Prometheus dashboards for live monitoring.

---

## Features

- **S3 Load Tests**: PUT, GET, DELETE, LIST operation load scenarios
- - **Throughput Benchmarking**: MB/s measurement across concurrency levels
  - - **Latency Profiling**: P50, P95, P99 latency distributions
    - - **Bottleneck Analysis**: Automated detection of throughput degradation points
      - - **Grafana Integration**: Pre-built dashboards for real-time metrics
        - - **Prometheus Exporter**: Custom metrics exporter for Locust stats
          - - **Ramp-up Scenarios**: Step-load, spike, soak, and stress test profiles
            - - **Multi-cluster**: Test multiple S3 endpoints simultaneously
             
              - ---

              ## Architecture

              ```
              performance-testing-locust/
              ├── locustfiles/
              │   ├── s3_put_load.py             # PUT operation load test
              │   ├── s3_get_load.py             # GET operation load test
              │   ├── s3_mixed_load.py           # Mixed read/write scenario
              │   └── s3_spike_test.py           # Spike load scenario
              ├── scenarios/
              │   ├── baseline_scenario.py       # Baseline performance test
              │   ├── stress_scenario.py         # Stress test to breaking point
              │   └── soak_scenario.py           # 4-hour soak test
              ├── monitoring/
              │   ├── grafana/
              │   │   └── s3-performance-dashboard.json  # Grafana dashboard JSON
              │   └── prometheus/
              │       └── prometheus.yml         # Prometheus scrape config
              ├── utils/
              │   ├── s3_client.py               # Boto3 S3 client wrapper
              │   ├── metrics_exporter.py        # Prometheus metrics exporter
              │   └── report_generator.py        # HTML report generator
              ├── config/
              │   └── perf_config.yaml           # Test configuration
              ├── reports/                       # Generated HTML reports
              └── README.md
              ```

              ---

              ## Quick Start

              ```bash
              # Install dependencies
              pip install locust boto3 prometheus-client pyyaml

              # Run S3 PUT load test (100 users, 10/s spawn rate)
              locust -f locustfiles/s3_put_load.py \
                --headless -u 100 -r 10 --run-time 5m \
                --host https://s3.swarm.internal \
                --html reports/put_load_report.html

              # Run mixed load test with Locust web UI
              locust -f locustfiles/s3_mixed_load.py \
                --host https://s3.swarm.internal

              # Run stress test
              locust -f scenarios/stress_scenario.py \
                --headless -u 500 -r 50 --run-time 10m \
                --html reports/stress_report.html
              ```

              ---

              ## Core Load Tests

              ### S3 PUT Load Test (`locustfiles/s3_put_load.py`)

              ```python
              import os
              import uuid
              import time
              import boto3
              import logging
              from locust import User, task, between, events
              from botocore.exceptions import ClientError

              logger = logging.getLogger(__name__)

              class S3PutUser(User):
                  """
                  Locust user that simulates S3 PUT operations against a Swarm cluster.
                  Measures throughput (MB/s) and latency for object uploads.
                  """
                  wait_time = between(0.1, 0.5)

                  def on_start(self):
                      """Initialise S3 client and create test bucket."""
                      self.client = boto3.client(
                          "s3",
                          endpoint_url=self.environment.host,
                          aws_access_key_id=os.environ.get("S3_ACCESS_KEY", "testkey"),
                          aws_secret_access_key=os.environ.get("S3_SECRET_KEY", "testsecret"),
                          verify=False
                      )
                      self.bucket = f"perf-test-{os.environ.get('TEST_RUN_ID', 'default')}"
                      self._ensure_bucket()

                  def _ensure_bucket(self):
                      try:
                          self.client.create_bucket(Bucket=self.bucket)
                      except ClientError as e:
                          if e.response["Error"]["Code"] != "BucketAlreadyOwnedByYou":
                              raise

                  @task(weight=1)
                  def put_small_object(self):
                      """Upload 64KB object and measure latency."""
                      self._put_object(size_bytes=64 * 1024, label="PUT_64KB")

                  @task(weight=1)
                  def put_medium_object(self):
                      """Upload 1MB object and measure throughput."""
                      self._put_object(size_bytes=1024 * 1024, label="PUT_1MB")

                  @task(weight=1)
                  def put_large_object(self):
                      """Upload 10MB object for throughput benchmarking."""
                      self._put_object(size_bytes=10 * 1024 * 1024, label="PUT_10MB")

                  def _put_object(self, size_bytes: int, label: str):
                      key = f"perf/{label}/{uuid.uuid4().hex}.bin"
                      data = os.urandom(size_bytes)
                      start_time = time.time()
                      try:
                          self.client.put_object(Bucket=self.bucket, Key=key, Body=data)
                          elapsed_ms = (time.time() - start_time) * 1000
                          throughput_mbps = (size_bytes / (1024 * 1024)) / (elapsed_ms / 1000)
                          self.environment.events.request.fire(
                              request_type="S3",
                              name=label,
                              response_time=elapsed_ms,
                              response_length=size_bytes,
                              exception=None,
                              context={"throughput_mbps": throughput_mbps}
                          )
                      except Exception as e:
                          elapsed_ms = (time.time() - start_time) * 1000
                          self.environment.events.request.fire(
                              request_type="S3",
                              name=label,
                              response_time=elapsed_ms,
                              response_length=0,
                              exception=e,
                              context={}
                          )
              ```

              ### Mixed Load Test (`locustfiles/s3_mixed_load.py`)

              ```python
              import os
              import uuid
              import time
              import boto3
              from locust import User, task, between

              class S3MixedUser(User):
                  """
                  Mixed read/write workload: 70% GET, 20% PUT, 10% DELETE.
                  Simulates realistic production traffic patterns on Swarm clusters.
                  """
                  wait_time = between(0.05, 0.2)

                  def on_start(self):
                      self.s3 = boto3.client(
                          "s3",
                          endpoint_url=self.environment.host,
                          aws_access_key_id=os.environ.get("S3_ACCESS_KEY"),
                          aws_secret_access_key=os.environ.get("S3_SECRET_KEY"),
                          verify=False
                      )
                      self.bucket = "perf-mixed-workload"
                      self.available_keys = []
                      self._seed_objects(count=100)

                  def _seed_objects(self, count: int):
                      """Pre-populate bucket with objects for GET tests."""
                      for _ in range(count):
                          key = f"seed/{uuid.uuid4().hex}.bin"
                          self.s3.put_object(Bucket=self.bucket, Key=key, Body=os.urandom(1024 * 1024))
                          self.available_keys.append(key)

                  @task(weight=7)
                  def get_object(self):
                      """GET operation — 70% of workload."""
                      if not self.available_keys:
                          return
                      key = self.available_keys[hash(uuid.uuid4()) % len(self.available_keys)]
                      start = time.time()
                      try:
                          response = self.s3.get_object(Bucket=self.bucket, Key=key)
                          response["Body"].read()
                          elapsed = (time.time() - start) * 1000
                          self.environment.events.request.fire(
                              request_type="S3", name="GET_1MB",
                              response_time=elapsed, response_length=1024*1024, exception=None, context={}
                          )
                      except Exception as e:
                          self.environment.events.request.fire(
                              request_type="S3", name="GET_1MB",
                              response_time=(time.time()-start)*1000, response_length=0, exception=e, context={}
                          )

                  @task(weight=2)
                  def put_object(self):
                      """PUT operation — 20% of workload."""
                      key = f"load/{uuid.uuid4().hex}.bin"
                      start = time.time()
                      try:
                          self.s3.put_object(Bucket=self.bucket, Key=key, Body=os.urandom(1024*1024))
                          self.available_keys.append(key)
                          elapsed = (time.time() - start) * 1000
                          self.environment.events.request.fire(
                              request_type="S3", name="PUT_1MB",
                              response_time=elapsed, response_length=1024*1024, exception=None, context={}
                          )
                      except Exception as e:
                          self.environment.events.request.fire(
                              request_type="S3", name="PUT_1MB",
                              response_time=(time.time()-start)*1000, response_length=0, exception=e, context={}
                          )

                  @task(weight=1)
                  def delete_object(self):
                      """DELETE operation — 10% of workload."""
                      if not self.available_keys:
                          return
                      key = self.available_keys.pop(0)
                      start = time.time()
                      try:
                          self.s3.delete_object(Bucket=self.bucket, Key=key)
                          elapsed = (time.time() - start) * 1000
                          self.environment.events.request.fire(
                              request_type="S3", name="DELETE",
                              response_time=elapsed, response_length=0, exception=None, context={}
                          )
                      except Exception as e:
                          self.environment.events.request.fire(
                              request_type="S3", name="DELETE",
                              response_time=(time.time()-start)*1000, response_length=0, exception=e, context={}
                          )
              ```

              ---

              ## Benchmark Results (Ragnarok Cluster)

              Results from WARP + Locust benchmarks on DataCore Swarm Ragnarok cluster:

              | Operation | Object Size | Concurrency | Throughput | Avg Latency | P99 Latency |
              |-----------|------------|-------------|------------|-------------|-------------|
              | PUT | 1 MB | 32 | 487 MB/s | 65 ms | 312 ms |
              | PUT | 10 MB | 16 | 623 MB/s | 252 ms | 890 ms |
              | GET | 1 MB | 64 | 1,240 MB/s | 51 ms | 198 ms |
              | GET | 10 MB | 32 | 1,890 MB/s | 168 ms | 445 ms |
              | DELETE | - | 100 | 2,450 ops/s | 41 ms | 122 ms |

              ---

              ## Grafana Dashboard

              Pre-built Grafana dashboard includes:

              - **Throughput over time** (MB/s, ops/s)
              - - **Latency percentiles** (P50, P95, P99)
                - - **Error rate** and failure breakdown
                  - - **Concurrent user count** timeline
                    - - **Per-operation breakdown** (PUT vs GET vs DELETE)
                     
                      - ---

                      ## Author

                      **Anuj Gupta** — Lead QA Automation Engineer / Staff SDET
                      [LinkedIn](https://linkedin.com/in/anuj-gupta-sdet) | anuj180993@gmail.com
