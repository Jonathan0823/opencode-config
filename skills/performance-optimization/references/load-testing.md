# Load Testing

## k6 Load Testing

### Basic Script

```javascript
// load-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';

// Test configuration
export const options = {
  stages: [
    { duration: '2m', target: 100 },   // Ramp up
    { duration: '5m', target: 100 },   // Stay at 100
    { duration: '2m', target: 200 },   // Ramp to 200
    { duration: '5m', target: 200 },   // Stay at 200
    { duration: '2m', target: 0 },     // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'],   // 95% under 500ms
    http_req_failed: ['rate<0.01'],     // Error rate < 1%
  },
};

export default function () {
  const res = http.get('https://api.example.com/users');
  
  check(res, {
    'status is 200': (r) => r.status === 200,
    'response time < 500ms': (r) => r.timings.duration < 500,
  });
  
  sleep(1);
}
```

### Advanced k6

```javascript
import http from 'k6/http';
import { check, group, sleep } from 'k6';
import { Rate, Trend } from 'k6/metrics';

// Custom metrics
const errorRate = new Rate('errors');
const responseTime = new Trend('response_time');

export const options = {
  scenarios: {
    smoke: {
      executor: 'constant-vus',
      vus: 1,
      duration: '1m',
      tags: { test_type: 'smoke' },
    },
    load: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '5m', target: 100 },
        { duration: '10m', target: 100 },
        { duration: '5m', target: 200 },
        { duration: '10m', target: 200 },
        { duration: '5m', target: 0 },
      ],
      tags: { test_type: 'load' },
    },
    stress: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '2m', target: 100 },
        { duration: '5m', target: 100 },
        { duration: '2m', target: 200 },
        { duration: '5m', target: 200 },
        { duration: '2m', target: 300 },
        { duration: '5m', target: 300 },
        { duration: '2m', target: 400 },
        { duration: '5m', target: 400 },
        { duration: '5m', target: 0 },
      ],
      tags: { test_type: 'stress' },
    },
  },
  thresholds: {
    http_req_duration: ['p(95)<500'],
    http_req_failed: ['rate<0.01'],
    errors: ['rate<0.05'],
  },
};

export function setup() {
  // Login and get token
  const loginRes = http.post('https://api.example.com/login', {
    email: 'test@example.com',
    password: 'password',
  });
  
  return { token: loginRes.json('token') };
}

export default function (data) {
  const params = {
    headers: {
      'Authorization': `Bearer ${data.token}`,
      'Content-Type': 'application/json',
    },
  };
  
  group('User Flow', () => {
    // Get user profile
    const profileRes = http.get('https://api.example.com/profile', params);
    
    check(profileRes, {
      'profile status is 200': (r) => r.status === 200,
    }) || errorRate.add(1);
    
    responseTime.add(profileRes.timings.duration);
    
    // Get orders
    const ordersRes = http.get('https://api.example.com/orders', params);
    
    check(ordersRes, {
      'orders status is 200': (r) => r.status === 200,
      'orders loaded in < 1s': (r) => r.timings.duration < 1000,
    }) || errorRate.add(1);
    
    sleep(Math.random() * 3 + 1);  // 1-4s think time
  });
}

export function teardown(data) {
  // Cleanup
  http.post('https://api.example.com/logout', null, {
    headers: { 'Authorization': `Bearer ${data.token}` },
  });
}
```

### Running k6

```bash
# Basic run
k6 run load-test.js

# With environment variables
k6 run -e BASE_URL=https://staging.example.com load-test.js

# Cloud execution
k6 cloud run load-test.js

# Output to InfluxDB
k6 run --out influxdb=http://localhost:8086/k6 load-test.js
```

## Artillery

### Basic Test

```yaml
# test.yml
config:
  target: 'https://api.example.com'
  phases:
    - duration: 60
      arrivalRate: 10
    - duration: 120
      arrivalRate: 50
  defaults:
    headers:
      Content-Type: 'application/json'

scenarios:
  - name: 'Get users'
    weight: 70
    requests:
      - get:
          url: '/users'
          capture:
            - json: '$.data[0].id'
              as: 'userId'

  - name: 'Create user'
    weight: 30
    requests:
      - post:
          url: '/users'
          json:
            name: 'Test User'
            email: 'test@example.com'
```

### Advanced Artillery

```yaml
# advanced-test.yml
config:
  target: 'https://api.example.com'
  phases:
    - name: 'Warm up'
      duration: 60
      arrivalRate: 5
    - name: 'Ramp up'
      duration: 120
      arrivalRate: 5
      rampTo: 50
    - name: 'Peak load'
      duration: 300
      arrivalRate: 50
    - name: 'Ramp down'
      duration: 60
      arrivalRate: 50
      rampTo: 5
  
  payload:
    - path: 'users.csv'
      fields:
        - 'email'
        - 'password'
  
  plugins:
    expect: {}

scenarios:
  - name: 'Login and browse'
    beforeRequest: 'setAuthHeader'
    requests:
      - post:
          url: '/login'
          json:
            email: '{{ email }}'
            password: '{{ password }}'
          capture:
            - json: '$.token'
              as: 'token'
          expect:
            - statusCode: 200
      
      - get:
          url: '/dashboard'
          headers:
            Authorization: 'Bearer {{ token }}'
          expect:
            - statusCode: 200
            - contentType: json
      
      - think: 3
      
      - get:
          url: '/orders'
          headers:
            Authorization: 'Bearer {{ token }}'
```

### Custom Functions

```javascript
// functions.js
module.exports = {
  setAuthHeader: (requestParams, context, ee, next) => {
    if (context.vars.token) {
      requestParams.headers = requestParams.headers || {};
      requestParams.headers.Authorization = `Bearer ${context.vars.token}`;
    }
    return next();
  },
  
  generateRandomEmail: (userContext, events, done) => {
    const random = Math.random().toString(36).substring(7);
    userContext.vars.email = `user${random}@example.com`;
    return done();
  }
};
```

## Apache JMeter

### Test Plan Structure

```xml
<!-- test-plan.jmx -->
<TestPlan guiclass="TestPlanGui" testname="Load Test">
  <hashTree>
    <ThreadGroup guiclass="ThreadGroupGui" testname="Users">
      <stringProp name="ThreadGroup.num_threads">100</stringProp>
      <stringProp name="ThreadGroup.ramp_time">60</stringProp>
      <stringProp name="ThreadGroup.duration">300</stringProp>
      
      <hashTree>
        <HTTPSamplerProxy guiclass="HttpTestSampleGui" testname="Get Users">
          <stringProp name="HTTPSampler.domain">api.example.com</stringProp>
          <stringProp name="HTTPSampler.port">443</stringProp>
          <stringProp name="HTTPSampler.protocol">https</stringProp>
          <stringProp name="HTTPSampler.path">/users</stringProp>
          <stringProp name="HTTPSampler.method">GET</stringProp>
        </HTTPSamplerProxy>
        
        <hashTree>
          <ResponseAssertion guiclass="AssertionGui" testname="Status 200">
            <collectionProp name="Asserion.test_strings">
              <stringProp name="49586">200</stringProp>
            </collectionProp>
            <stringProp name="Assertion.test_field">Assertion.response_code</stringProp>
          </ResponseAssertion>
        </hashTree>
      </hashTree>
    </ThreadGroup>
    
    <ResultCollector guiclass="ViewResultsFullVisualizer" testname="View Results Tree"/>
  </hashTree>
</TestPlan>
```

### Running JMeter

```bash
# GUI mode
jmeter -t test-plan.jmx

# CLI mode
jmeter -n -t test-plan.jmx -l results.jtl -e -o report/

# Distributed mode
jmeter -n -t test-plan.jmx -R server1,server2
```

## Locust

### Python Load Test

```python
# locustfile.py
from locust import HttpUser, task, between
import random

class WebsiteUser(HttpUser):
    wait_time = between(1, 5)
    
    def on_start(self):
        """Login on start."""
        response = self.client.post('/login', json={
            'email': 'test@example.com',
            'password': 'password'
        })
        self.token = response.json()['token']
    
    @task(3)
    def get_users(self):
        self.client.get('/users', headers={
            'Authorization': f'Bearer {self.token}'
        })
    
    @task(2)
    def get_user_detail(self):
        user_id = random.randint(1, 1000)
        self.client.get(f'/users/{user_id}', headers={
            'Authorization': f'Bearer {self.token}'
        })
    
    @task(1)
    def create_order(self):
        self.client.post('/orders', json={
            'product_id': random.randint(1, 100),
            'quantity': random.randint(1, 5)
        }, headers={
            'Authorization': f'Bearer {self.token}'
        })
```

### Running Locust

```bash
# Web UI
locust -f locustfile.py --host=https://api.example.com

# Headless
locust -f locustfile.py --headless -u 100 -r 10 --run-time 5m

# Distributed
locust -f locustfile.py --master
locust -f locustfile.py --worker --master-host=localhost
```

## Test Types

### Smoke Test
```yaml
# Quick verification
duration: 1m
vus: 1-5
assertions:
  - error_rate < 1%
  - p95 < 1000ms
```

### Load Test
```yaml
# Expected normal load
duration: 30m
vus: 100-500
ramp_up: 5m
steady_state: 20m
ramp_down: 5m
```

### Stress Test
```yaml
# Find breaking point
duration: 1h
vus: 100 → 1000 (incremental)
stop_on_error_rate: 50%
```

### Spike Test
```yaml
# Sudden traffic spike
duration: 10m
vus: 10 → 1000 → 10
spike_duration: 1m
```

### Soak Test
```yaml
# Long-running test
duration: 24h
vus: 100
monitor: memory_leaks, connection_pool_exhaustion
```

## CI/CD Integration

```yaml
# .github/workflows/load-test.yml
name: Load Test

on:
  schedule:
    - cron: '0 2 * * 0'  # Weekly on Sunday
  workflow_dispatch:

jobs:
  load-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Run k6
        uses: grafana/k6-action@v0.3.1
        with:
          filename: load-test.js
          flags: --out json=results.json
      
      - name: Upload results
        uses: actions/upload-artifact@v3
        with:
          name: load-test-results
          path: results.json
      
      - name: Check thresholds
        run: |
          if grep -q "failed" results.json; then
            echo "Load test failed thresholds"
            exit 1
          fi
```
