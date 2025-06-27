# lerning jaeger

## ตัวอย่าง Log Tracing ด้วย Jaeger ร่วมกับ Node.js และ Express.js

Jaeger เป็นระบบ Open Source ที่ใช้สำหรับติดตาม (tracing) ธุรกรรมที่กระจายตัว (distributed transactions) ซึ่งมีประโยชน์อย่างยิ่งในการตรวจสอบและแก้ไขปัญหาใน microservices architecture หรือแม้แต่ application เดียวที่มีความซับซ้อน

นี่คือขั้นตอนการทำ Log tracing ด้วย Jaeger ร่วมกับ Node.js และ Express.js แบบทีละขั้นตอน:

**1. ตั้งค่า Jaeger Infrastructure (Local Development):**

สำหรับการพัฒนาบนเครื่องของคุณ คุณสามารถใช้ Docker เพื่อรัน Jaeger ได้อย่างง่ายดาย:

เปิด Terminal แล้วรันคำสั่งนี้:

```bash
docker run -d --name jaeger \
  -p 5775:5775/udp \
  -p 6831:6831/udp \
  -p 6832:6832/udp \
  -p 5778:5778 \
  -p 16686:16686 \
  -p 14268:14268 \
  -p 14250:14250 \
  jaegertracing/all-in-one:latest
```

  * **5775/udp, 6831/udp, 6832/udp:** พอร์ตสำหรับส่ง spans ในรูปแบบต่างๆ
  * **5778:** พอร์ตสำหรับ health check
  * **16686:** พอร์ตสำหรับ Jaeger UI (เปิดใน Browser เพื่อดู traces)
  * **14268:** พอร์ตสำหรับ Jaeger gRPC collector
  * **14250:** พอร์ตสำหรับ Jaeger agent health check

เมื่อรัน Docker container สำเร็จ คุณสามารถเข้าถึง Jaeger UI ได้ที่ [http://localhost:16686](https://www.google.com/search?q=http://localhost:16686)

**2. สร้าง Project Node.js และติดตั้ง Dependencies:**

หากคุณมี Project Express.js อยู่แล้ว สามารถข้ามขั้นตอนนี้ได้

1.  สร้างโฟลเดอร์สำหรับ Project:

    ```bash
    mkdir express-jaeger-example
    cd express-jaeger-example
    ```

2.  Initial Project Node.js:

    ```bash
    npm init -y
    ```

3.  ติดตั้ง Express.js และ Jaeger Client Library:
    เราจะใช้ Library ที่แนะนำโดย Jaeger ซึ่งคือ `jaeger-client`:

    ```bash
    npm install express jaeger-client opentracing
    ```

    หรือคุณอาจพิจารณาใช้ `opentelemetry` ซึ่งเป็นมาตรฐานใหม่และรองรับ vendor อื่นๆ นอกเหนือจาก Jaeger หากต้องการใช้ `opentelemetry` แทน `jaeger-client` คุณจะต้องติดตั้ง dependencies ที่เกี่ยวข้องกับ OpenTelemetry และ Jaeger exporter แทน

**3. ตั้งค่า Jaeger Client ใน Application:**

สร้างไฟล์ เช่น `tracing.js` เพื่อตั้งค่า Jaeger client:

```javascript
const { initTracer } = require('jaeger-client');
const opentracing = require('opentracing');

function init(serviceName) {
  const config = {
    serviceName: serviceName,
    sampler: {
      type: 'const',
      param: 1, // Sample every trace. Adjust for production (e.g., 0.1 for 10%)
    },
    reporter: {
      logSpans: true,
      agentHost: 'localhost', // Default Jaeger agent host
      agentPort: 6832,       // Default Jaeger agent port for UDP
    },
  };
  const options = {
    logger: {
      info: (msg) => console.log('INFO ', msg),
      error: (msg) => console.log('ERROR', msg),
    },
  };
  const tracer = initTracer(config, options);
  opentracing.initGlobalTracer(tracer);
  return tracer;
}

module.exports.initTracer = init;
```

  * `serviceName`: ชื่อของ service ของคุณ (เช่น `user-service`, `order-service`) ซึ่งจะปรากฏใน Jaeger UI
  * `sampler`: กำหนดกลยุทธ์การสุ่มตัวอย่าง (sampling) สำหรับ traces ใน Production คุณอาจต้องการสุ่มตัวอย่างเพียงบางส่วนเพื่อลดปริมาณข้อมูล
  * `reporter`: กำหนดวิธีการรายงาน trace data ไปยัง Jaeger agent หรือ collector
      * `logSpans: true`: จะ log spans ไปยัง console (มีประโยชน์สำหรับการพัฒนา)
      * `agentHost`, `agentPort`: ที่อยู่และพอร์ตของ Jaeger agent (ค่า default คือ `localhost:6832`)

**4. Instrument Application Express.js:**

แก้ไขไฟล์หลักของ Application (เช่น `app.js` หรือ `server.js`) เพื่อ Instrument การทำงาน:

```javascript
const express = require('express');
const app = express();
const port = 3000;
const { initTracer } = require('./tracing');
const opentracing = require('opentracing');

// Initialize Jaeger Tracer
const tracer = initTracer('express-app');

// Middleware เพื่อสร้าง Span สำหรับแต่ละ Request
app.use((req, res, next) => {
  const span = tracer.startSpan(req.path);
  span.setTag('http.method', req.method);
  span.setTag('http.url', req.originalUrl);
  span.setTag('span.kind', 'server'); // บ่งบอกว่าเป็น Server Span

  // Inject Span Context เข้าไปใน Request เพื่อให้ Span ลูกสามารถอ้างอิงได้
  const carrier = {};
  tracer.inject(span.context(), opentracing.FORMAT_HTTP_HEADERS, carrier);
  req.headers['uber-trace-id'] = carrier['uber-trace-id']; // ส่งต่อ Trace ID (หากต้องการ)

  res.on('finish', () => {
    span.setTag('http.status_code', res.statusCode);
    span.finish();
  });

  next();
});

app.get('/', (req, res) => {
  // สร้าง Child Span (Optional)
  const parentSpanContext = opentracing.globalTracer().extract(opentracing.FORMAT_HTTP_HEADERS, req.headers);
  const childSpan = tracer.startSpan('handle-root', { childOf: parentSpanContext });
  childSpan.log({ event: 'processing request' });
  setTimeout(() => {
    childSpan.log({ event: 'request processed' });
    childSpan.finish();
    res.send('Hello World!');
  }, 100);
});

app.get('/users/:id', (req, res) => {
  const userId = req.params.id;
  const span = opentracing.globalTracer().startSpan('get-user', { childOf: opentracing.globalTracer().getCurrentSpan().context() }); // สร้าง Child Span ภายใน Middleware Span
  span.setTag('user.id', userId);
  setTimeout(() => {
    span.finish();
    res.send(`User ID: ${userId}`);
  }, 50);
});

app.listen(port, () => {
  console.log(`Example app listening on port ${port}`);
});
```

  * เรา Initial Jaeger tracer ด้วยชื่อ Service ของ Application
  * เราสร้าง Middleware ที่จะทำงานกับทุกๆ Request ที่เข้ามา
      * ใน Middleware เราสร้าง Span ใหม่สำหรับแต่ละ Request โดยใช้ `tracer.startSpan(req.path)`
      * เราตั้ง Tags ที่เกี่ยวข้องกับ HTTP Request (method, URL, status code)
      * เรา Inject Span Context เข้าไปใน Headers ของ Request (`uber-trace-id`) ซึ่งมีประโยชน์ถ้า Application นี้เรียก Services อื่นๆ
      * เมื่อ Response เสร็จสิ้น (`res.on('finish')`) เราจะตั้ง Tag `http.status_code` และเรียก `span.finish()` เพื่อปิด Span
  * ใน Route Handlers (`/` และ `/users/:id`) เราสามารถสร้าง Child Spans เพิ่มเติมเพื่อติดตามการทำงานภายใน Handler นั้นๆ

**5. ทดสอบ Application:**

1.  รัน Application Node.js:

    ```bash
    node app.js
    ```

2.  เรียก API ของคุณผ่าน Browser หรือเครื่องมืออื่นๆ เช่น `curl`:

      * `http://localhost:3000/`
      * `http://localhost:3000/users/123`

**6. ดูผลลัพธ์ใน Jaeger UI:**

1.  เปิด Browser ไปที่ [http://localhost:16686](https://www.google.com/search?q=http://localhost:16686)
2.  เลือก `express-app` จาก Dropdown ของ `Service`
3.  คลิกที่ปุ่ม `Find Traces`

คุณจะเห็นรายการของ Traces ที่ถูกสร้างขึ้นสำหรับแต่ละ Request ที่คุณส่งเข้ามา

คลิกที่ Trace ใด Trace หนึ่งเพื่อดูรายละเอียดของ Spans ที่เกี่ยวข้อง คุณจะเห็น:

  * **Trace ID:** ID ที่เป็นเอกลักษณ์ของ Request นั้นๆ
  * **Spans:** แต่ละขั้นตอนการทำงานที่ถูก Instrument (เช่น Request Middleware, Handler ของ Route `/`, Child Span `handle-root`, Child Span `get-user`)
  * **Operation Name:** ชื่อของ Span (เช่น ชื่อ Route path)
  * **Duration:** ระยะเวลาที่ Span นั้นใช้
  * **Tags:** ข้อมูลเพิ่มเติมที่ถูกบันทึกไว้ใน Span (เช่น `http.method`, `http.url`, `user.id`)
  * **Logs:** เหตุการณ์ที่เกิดขึ้นภายใน Span พร้อม Timestamp

**7. (Optional) การส่งต่อ Trace Context ไปยัง Services อื่นๆ:**

หาก Application ของคุณมีการเรียก Services อื่นๆ คุณจะต้องส่งต่อ Trace Context (โดยเฉพาะ `uber-trace-id` หรือรูปแบบอื่นๆ ที่ Jaeger หรือ OpenTelemetry รองรับ) ไปใน HTTP Headers ของ Request ที่จะส่งไปยัง Services เหล่านั้น

ใน Service ที่ถูกเรียก คุณจะต้องใช้ Jaeger client หรือ OpenTelemetry SDK เพื่อ Extract Trace Context จาก Headers และสร้าง Child Spans ที่เชื่อมโยงกับ Trace เดิม

**ข้อควรจำ:**

  * ใน Production คุณควรพิจารณาการตั้งค่า `sampler` ให้เหมาะสมเพื่อลดปริมาณข้อมูล Trace ที่ส่งไปยัง Jaeger
  * คุณสามารถเพิ่ม Tags และ Logs ที่มีความหมายลงใน Spans เพื่อให้การติดตามง่ายขึ้น
  * สำหรับ Asynchronous Operations (เช่น การเรียกฐานข้อมูล, การทำงานกับ Message Queues) คุณจะต้องสร้าง Child Spans และจัดการ Context อย่างเหมาะสมเพื่อให้ Trace ยังคงถูกต้อง
  * นอกจาก `jaeger-client` แล้ว ยังมี Libraries อื่นๆ ที่รองรับ OpenTracing หรือ OpenTelemetry ซึ่งคุณสามารถเลือกใช้ได้ตามความเหมาะสม

การทำ Log tracing ด้วย Jaeger จะช่วยให้คุณมีความเข้าใจลึกซึ้งเกี่ยวกับการทำงานของ Application ของคุณ ทำให้ง่ายต่อการระบุและแก้ไขปัญหาที่อาจเกิดขึ้นในการทำงานจริง
