# Steps

### open yegar
```bash
docker run -d --name jaeger -e COLLECTOR_ZIPKIN_HOST_PORT=:9411 -p 5775:5775/udp -p 6831:6831/udp -p 6832:6832/udp -p 5778:5778 -p 16686:16686 -p 14268:14268 -p 14250:14250 -p 9411:9411 jaegertracing/all-in-one:1.23

# then go to `http://localhost:16686/search`
```

### install dependency
```bash
npm install @opentelemetry/api @opentelemetry/sdk-node @opentelemetry/auto-instrumentations-node @opentelemetry/exporter-trace-otlp-http @opentelemetry/exporter-jaeger

npm install @opentelemetry/instrumentation-mongodb @opentelemetry/instrumentation-ioredis
```

### copy tracing.js into the directory
```js
const { NodeTracerProvider } = require("@opentelemetry/node");
const { registerInstrumentations } = require("@opentelemetry/instrumentation");
const {
  ExpressInstrumentation,
} = require("@opentelemetry/instrumentation-express");
const { JaegerExporter } = require("@opentelemetry/exporter-jaeger");
const { BatchSpanProcessor } = require("@opentelemetry/tracing");
const { Resource } = require("@opentelemetry/resources");
const {
  SemanticResourceAttributes,
} = require("@opentelemetry/semantic-conventions");
const {
  RedisInstrumentation,
} = require("@opentelemetry/instrumentation-redis");
function configureOpenTelemetry(serviceName) {
  // Create a tracer provider and register the Express instrumentation
  const provider = new NodeTracerProvider({
    resource: new Resource({
      [SemanticResourceAttributes.SERVICE_NAME]: serviceName,
      // Add other resource attributes as needed
    }),
  });
  provider.register();

  // Configure and register Jaeger exporter
  const exporter = new JaegerExporter({
    serviceName: serviceName,
    agentHost: "localhost", // Change this to your Jaeger host
    agentPort: 16686, // Change this to your Jaeger port
  });

  // Use BatchSpanProcessor
  const spanProcessor = new BatchSpanProcessor(exporter);
  provider.addSpanProcessor(spanProcessor);

  // Register the Express instrumentation
  registerInstrumentations({
    tracerProvider: provider,
    instrumentations: [
      new ExpressInstrumentation(),
      new RedisInstrumentation(),
    ],
  });

  return provider;
}

module.exports = configureOpenTelemetry;
```

### in the node app add following initiators
```js
const configureOpenTelemetry = require("./tracing");
const { trace, context, propagation } = require("@opentelemetry/api");
const tracerProvider = configureOpenTelemetry("service-name"); // add service name here

app.use((req, res, next) => {
  const tracer = tracerProvider.getTracer("express-tracer");
  const span = tracer.startSpan("parent-span");

  // Add custom attributes or log additional information if needed
  span.setAttribute("team name", "SUST Define Coders");

  // Pass the span to the request object for use in the route handler
  context.with(trace.setSpan(context.active(), span), () => {
    next();
  });
});
```

### in the api itself do following
```js
    const parentSpan = trace.getSpan(context.active());
    // can add data to parent span use try catch
    if (parentSpan) {
      parentSpan.setAttribute("user.id", user.id);
      parentSpan.setAttribute("user.name", user.name);
    }        
```

### in the axios call do the following for context propagation
```js
    try {    

    const respose_data = await context.with(
      trace.setSpan(context.active(), parentSpan),
      async () => {        
        const carrier = {};
        propagation.inject(context.active(), carrier);

        // replace axios call here
        return axios.get("http://localhost:5000/validateuser", {
          headers: carrier,
        });
      }
    );

    console.log("Validation response:", respose_data.data); // Log or use the response as needed
    
    // add what you want as response
    res.json(user);

  } catch (error) {
    if (parentSpan) {
      parentSpan.recordException(error);
    }
    res.status(500).send(error.message);
  } finally {
    if (parentSpan) {
      parentSpan.end();
    }
  }
```

### add shutdown code
```js
// Gracefully shut down the OpenTelemetry SDK and the server
const gracefulShutdown = () => {
  server.close(() => {
    console.log("Server stopped");
    sdk
      .shutdown()
      .then(() => console.log("Tracing terminated"))
      .catch((error) => console.error("Error shutting down tracing", error))
      .finally(() => process.exit(0));
  });
};

// Listen for termination signals
process.on("SIGTERM", gracefulShutdown);
process.on("SIGINT", gracefulShutdown);
```

### in the network first do the setup again
```js
// setup
const configureOpenTelemetry = require("./tracing");
configureOpenTelemetry("service name");
const { context, trace, propagation } = require("@opentelemetry/api");
```

### in the api do:
```js
  const ctx = propagation.extract(context.active(), req.headers); 
  const tracer = trace.getTracer("express-tracer");
  console.log("Incoming request headers:", req.headers);
  console.log(
    "Extracted span from context:",
    trace.getSpan(ctx)?.spanContext()
  ); // Retrieve span from extracted context

  const span = tracer.startSpan(
    "new-child-span-name",
    {
      attributes: { "http.method": "GET", "http.url": req.url },
    },
    ctx
  );
  span.end();
  res.json({ success: true });
```

### create a child span
```js
  const tracer = tracerProvider.getTracer("express-tracer");
  const childSpan = tracer.startSpan("validation", { parent: parentSpan });
  // code here
  childSpan.end();
```

### to get trace id
```js
  const traceId = parentSpan.spanContext().traceId;
  res.send(traceId, 200);
```

