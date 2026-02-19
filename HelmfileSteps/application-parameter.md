

## 1. **Add Network Policy for Your Application Pod**

First, ensure your application pod has the correct labels and network policies:

```yaml
# application-deployment.yaml (excerpt)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: your-application
  namespace: flows-pen-test
spec:
  template:
    metadata:
      labels:
        app: your-application  # This is important for network policies
    spec:
      containers:
      - name: your-app
        image: your-image:tag
        # ... rest of your container config
```

## 2. **Configure HTTP_PROXY Environment Variables**

Add environment variables to your application pod to route traffic through HAProxy:

```yaml
# In your application deployment YAML
spec:
  template:
    spec:
      containers:
      - name: your-app
        env:
        - name: HTTP_PROXY
          value: "http://haproxy-egress:8443"
        - name: HTTPS_PROXY
          value: "http://haproxy-egress:8443"
        - name: NO_PROXY
          value: "localhost,127.0.0.1,.cluster.local,10.0.0.0/8"  # Adjust as needed
```

## 3. **Alternative: Configure Application-Specific Proxy Settings**

If your application uses specific HTTP clients, you might need to configure them explicitly:

```yaml
# For Java applications
env:
- name: JAVA_TOOL_OPTIONS
  value: "-Dhttp.proxyHost=haproxy-egress -Dhttp.proxyPort=8443 -Dhttps.proxyHost=haproxy-egress -Dhttps.proxyPort=8443"

# For Python requests library
# This would need to be handled in code:
# import os
# os.environ['HTTP_PROXY'] = 'http://haproxy-egress:8443'
# os.environ['HTTPS_PROXY'] = 'http://haproxy-egress:8443'
```

## 4. **Add Diagnostic Container for Testing (Optional)**

Add an init container or sidecar for testing connectivity:

```yaml
# Add as an init container for pre-flight checks
initContainers:
- name: test-connectivity
  image: curlimages/curl:latest
  command:
    - /bin/sh
    - -c
    - |
      echo "Testing connectivity through HAProxy..."
      curl -v -x http://haproxy-egress:8443 https://r1r2.3disystems.com
      if [ $? -eq 0 ]; then
        echo "Connectivity test successful!"
      else
        echo "Connectivity test failed!"
        exit 1
      fi

# Or as a sidecar for ongoing testing
sidecarContainers:
- name: connectivity-tester
  image: curlimages/curl:latest
  command:
    - /bin/sh
    - -c
    - |
      while true; do
        echo "$(date): Testing connection..."
        curl -s -o /dev/null -w "%{http_code}\n" -x http://haproxy-egress:8443 https://r1r2.3disystems.com
        sleep 60
      done
```

## 5. **Complete Application YAML Example**

Here's a complete example incorporating all the above:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-app
  namespace: flows-pen-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: test-app
  template:
    metadata:
      labels:
        app: test-app  # Important for network policies
    spec:
      # Test connectivity before main container starts
      initContainers:
      - name: connectivity-test
        image: curlimages/curl:latest
        command:
          - /bin/sh
          - -c
          - |
            echo "Testing mTLS connectivity through HAProxy..."
            curl -v -x http://haproxy-egress:8443 https://r1r2.3disystems.com
            if [ $? -eq 0 ]; then
              echo "✓ mTLS connection successful!"
            else
              echo "✗ mTLS connection failed!"
              exit 1
            fi
      
      containers:
      - name: main-app
        image: your-application:latest
        env:
        # Proxy environment variables
        - name: HTTP_PROXY
          value: "http://haproxy-egress:8443"
        - name: HTTPS_PROXY
          value: "http://haproxy-egress:8443"
        - name: NO_PROXY
          value: "localhost,127.0.0.1,.cluster.local,10.0.0.0/8"
        
        # Java specific (if using Java)
        - name: JAVA_TOOL_OPTIONS
          value: "-Dhttp.proxyHost=haproxy-egress -Dhttp.proxyPort=8443 -Dhttps.proxyHost=haproxy-egress -Dhttps.proxyPort=8443"
        
        # Your app-specific environment variables
        - name: TARGET_URL
          value: "https://r1r2.3disystems.com"
      
      # Sidecar for continuous monitoring
      - name: connectivity-monitor
        image: curlimages/curl:latest
        command:
          - /bin/sh
          - -c
          - |
            while true; do
              HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" -x http://haproxy-egress:8443 https://r1r2.3disystems.com)
              echo "$(date): External domain connectivity test - HTTP $HTTP_CODE"
              sleep 30
            done
```

## 6. **Test Connectivity Manually**

If you need to test without modifying your application, you can create a temporary test pod:

```bash
# Create a test pod with curl
kubectl run curl-test -n flows-pen-test --image=curlimages/curl --rm -it --restart=Never -- /bin/sh

# Inside the pod, test connectivity through HAProxy
curl -v -x http://haproxy-egress:8443 https://r1r2.3disystems.com

# Test with more details
curl -v -x http://haproxy-egress:8443 \
  -H "Host: r1r2.3disystems.com" \
  https://r1r2.3disystems.com
```

## 7. **Verify Network Policies**

Ensure your application pod has the necessary labels that match the network policies:

```bash
# Check if your app has the right labels
kubectl get pod your-app-pod -n flows-pen-test --show-labels

# If needed, add the label to match network policies
kubectl label pod your-app-pod -n flows-pen-test app=your-application --overwrite
```

The key points are:
1. Set `HTTP_PROXY`/`HTTPS_PROXY` environment variables to point to `haproxy-egress:8443`
2. Ensure your application pod has appropriate labels
3. Add test containers to verify connectivity
4. Use the service name `haproxy-egress` which resolves to the HAProxy service

This configuration will route all external HTTPS traffic through the HAProxy mTLS proxy, which will then connect to `r1r2.3disystems.com` with the proper client certificates.




## For a UI application to communicate with an external domain through your existing HAProxy mTLS proxy, you need to add three main configurations to the `application.yaml`:

## 1. **Proxy Environment Variables**

Add these environment variables to your UI application container to route traffic through the HAProxy service :

```yaml
# In your UI application deployment YAML
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ui-application
  namespace: flows-pen-test
spec:
  template:
    spec:
      containers:
      - name: ui-app
        # ... your existing container config
        env:
        - name: HTTP_PROXY
          value: "http://haproxy-egress:8443"
        - name: HTTPS_PROXY
          value: "http://haproxy-egress:8443"
        - name: NO_PROXY
          value: "localhost,127.0.0.1,.cluster.local,10.0.0.0/8,10.120.0.0/16,172.16.0.0/16"
        - name: http_proxy
          value: "http://haproxy-egress:8443"
        - name: https_proxy
          value: "http://haproxy-egress:8443"
        - name: no_proxy
          value: "localhost,127.0.0.1,.cluster.local,10.0.0.0/8,10.120.0.0/16,172.16.0.0/16"
```

**Note**: Include both uppercase and lowercase versions as different applications may expect different formats .

## 2. **Application-Specific Configuration**

Many UI applications need explicit endpoint configuration. Based on your setup, add:

```yaml
# Additional environment variables for UI-specific proxy config
env:
# For React/Vue/Angular apps built with webpack
- name: HTTPS_PROXY
  value: "http://haproxy-egress:8443"

# If your UI app uses a config file for endpoints
- name: EXTERNAL_API_URL
  value: "https://r1r2.3disystems.com"  # Your target domain

# For Node.js based UI apps
- name: GLOBAL_AGENT_HTTP_PROXY
  value: "http://haproxy-osegress:8443"
- name: GLOBAL_AGENT_HTTPS_PROXY
  value: "http://haproxy-egress:8443"
```

## 3. **Network Policy Labels**

Ensure your UI pod has the correct labels to work with your existing network policies :

```yaml
metadata:
  labels:
    app: ui-application  # This should match your app identifier
    # Add any other labels your network policies expect
```

The existing network policies you shared already:
- Allow pods in `flows-pen-test` namespace to egress to HAProxy (port 8443)
- Allow HAProxy to egress to external CIDR (13.91.0.38/32) on port 443

## 4. **Testing Configuration (Optional)**

Add a test init container to verify connectivity :

```yaml
initContainers:
- name: test-external-connectivity
  image: curlimages/curl:latest
  command:
    - /bin/sh
    - -c
    - |
      echo "Testing connection through HAProxy..."
      curl -v -x http://haproxy-egress:8443 https://r1r2.3disystems.com/health
      if [ $? -eq 0 ]; then
        echo "✓ Successfully connected to external domain"
      else
        echo "✗ Failed to connect to external domain"
        exit 1
      fi
```

## Complete Example

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ui-application
  namespace: flows-pen-test
  labels:
    app: ui-application
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ui-application
  template:
    metadata:
      labels:
        app: ui-application
    spec:
      initContainers:
      - name: connectivity-test
        image: curlimages/curl:latest
        command:
          - /bin/sh
          - -c
          - |
            curl -v -x http://haproxy-egress:8443 https://r1r2.3disystems.com
      
      containers:
      - name: ui-app
        image: your-ui-image:tag
        env:
        # Proxy configuration
        - name: HTTP_PROXY
          value: "http://haproxy-egress:8443"
        - name: HTTPS_PROXY
          value: "http://haproxy-egress:8443"
        - name: NO_PROXY
          value: "localhost,127.0.0.1,.cluster.local,10.120.0.0/16,172.16.0.0/16"
        
        # Application-specific
        - name: REACT_APP_API_URL
          value: "https://r1r2.3disystems.com"
        - name: NODE_EXTRA_CA_CERTS
          value: "/etc/ssl/certs/ca-certificates.crt"
```

The key points are:
1. **Proxy variables** point to `haproxy-egress:8443` 
2. **NO_PROXY** excludes internal cluster communication from going through the proxy 
3. **Application labels** ensure network policies apply correctly 
4. **Test connectivity** using init containers to validate before the main app starts 
