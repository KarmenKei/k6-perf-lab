# Lab 02 — k6 Performance Testing

## Зорилго

Энэ лабораторийн зорилго нь k6 ашиглан веб хүсэлтийн latency, throughput болон error rate-ийг хэмжих явдал юм. Ачааллыг 5, 30, 100 VU түвшинд туршиж, хэрэглэгчийн тоо өсөхөд системийн гүйцэтгэл хэрхэн өөрчлөгдөж байгааг харьцуулав.

## Test target

Load test-ийн бай:

```text
[https://test.k6.io](https://test.k6.io)
```

Зөвшөөрөгдсөн дадлагын сайт ашигласан бөгөөд зөвшөөрөлгүй бодит систем рүү тест ажиллуулаагүй.

## Environment

- k6 version:

```text
k6 v2.2.0
```

- Operating system: `[macOS]`
- Test date: `[2026-09-14]`
- Script: `script.js`
- Results directory: `results/`

## Basic test

Үндсэн тест дараах хэлбэрээр ажилласан:

```bash
k6 run --vus 5 --duration 1m script.js
k6 run --vus 30 --duration 1m script.js
k6 run --vus 100 --duration 1m script.js
```

Тест бүрийн бүтэн гаралт дараах файлуудад хадгалагдсан:

- `results/run-05vu.txt`
- `results/run-30vu.txt`
- `results/run-100vu.txt`

![5vu](results/screenshots/05.png)
![30vu](results/screenshots/30.png)
![100](results/screenshots/100.png)

## VU comparison

| VU | p90 latency | p95 latency | Throughput | Error rate |
|---:|---:|---:|---:|---:|
| 5 | 228.71 ms | 229.18 ms | 7.40 req/s | 0.00% |
| 30 | 229.43 ms | 231.38 ms | 45.47 req/s | 0.00% |
| 100 | 229.51 ms | 232.14 ms | 150.42 req/s | 0.00% |

`p90` болон `p95` нь `http_req_duration` хэмжүүрээс, throughput нь `http_reqs`-ийн секундэд ногдох утгаас, error rate нь `http_req_failed` хэмжүүртэй.

## Stages test

Stages тохиргоо:

```javascript
export const options = {
  stages: [
    { duration: "30s", target: 5 },
    { duration: "1m", target: 30 },
    { duration: "30s", target: 100 },
    { duration: "30s", target: 0 },
  ],
};
```

Stages тестийн бүтэн гаралт:

```text
results/run-stages.txt
```
![Stages Test](results/screenshots/stages.png)

Stages тестээр ачаалал аажмаар 5 -> 30 -> 100 VU болж өсөж, дараа нь 0 болж буурсан ерөнхий хэлбэрийг ажиглав. Нийт 7072 request, дундаж throughput 46.87 req/s, p95 latency 231.15 ms, error rate 0.00% байсан бөгөөд гурван түвшний тоон харьцуулалтыг stages-ийн нийлбэр summary-ээс бус, тусдаа 5, 30, 100 VU утгуудаар авав.

## SLO and thresholds

SLO-г туршилтын хамгийн бага ачаалал болох **5 VU-ийн baseline хэмжилтэд** үндэслэв. 5 VU нь системийн үндсэн гүйцэтгэлийг ачаалал харьцангуй бага үед хэмжих боломжтой тул baseline болгон сонгосон. Энэ үед `http_req_duration`-ийн p95 latency **229.18 ms** байна.

Baseline-ийн p95 утгыг үндэслэн ачаалал нэмэгдэх үед бага хэмжээний хэлбэлзэл гарах боломжийг тооцож, SLO-ийн latency босгыг baseline-ийн **232.43 ms** дээр тогтов.

* **Latency SLO:** `p95 < 229.18 ms`
* **Error rate SLO:** `< 1%`

Ингэснээр систем ачаалал нэмэгдэх үед baseline-тай харьцуулахад latency хэт их өсөхгүй, мөн хүсэлтийн алдааны түвшин 1%-иас бага байх шаардлагатай гэж үзсэн.
Threshold тохиргоо:

```javascript
thresholds: {
  http_req_duration: ["p(95)<408"],
  http_req_failed: ["rate<0.01"],
}
```

PASS үр дүн:

```text
results/run-threshold-pass.txt
```
![Threshold PASS](results/screenshots/succeed.png)


Санаатайгаар хатуу threshold ашигласан FAIL үр дүн:

```text
results/run-threshold-fail.txt
```
![Treshold FAIL](results/screenshots/fail.png)

PASS тестэд `p(95)<408` болон `rate<0.01` threshold хоёулаа хангагдсан (p95 = 233.58 ms). FAIL тестэд `p(95)<50` threshold хангагдаагүй (p95 = 232.43 ms) тул k6 тестийг амжилтгүй гэж тэмдэглэв.

## Дүгнэлт

5 VU-ийн үед системийн p95 latency **229.18 ms** байсан бөгөөд үүнийг baseline гүйцэтгэлийн үзүүлэлт болгон авсан. Ачааллыг 30 VU хүртэл нэмэгдүүлэхэд throughput **7.48 req/s-ээс 45.47 req/s** болж мэдэгдэхүйц өссөн бөгөөд p95 latency **229.18 ms-ээс 231.38 ms** болж бага зэрэг нэмэгдсэн. 100 VU-ийн үед throughput **150.42 req/s** хүрсэн бөгөөд p95 latency **232.14 ms** байсан. Иймээс энэхүү туршилтын хүрээнд хэрэглэгчийн тоо нэмэгдэхэд throughput мэдэгдэхүйц өссөн боловч latency огцом нэмэгдээгүй, харьцангуй тогтвортой хэвээр байв. Мөн 5, 30, 100 VU-ийн бүх түвшинд error rate **0.00%** байсан нь туршилтын явцад хүсэлтүүд алдаагүй боловсруулагдсаныг харуулж байна.  
  
Үүнээс throughput, latency болон error rate үзүүлэлтүүдийг ашиглан системийн ачаалал нэмэгдэх үеийн гүйцэтгэлийн өөрчлөлтийг бодитоор хэмжинэ. Stages тестээр ачааллыг **5 -> 30 ->100 VU** болгон өсгөж, дараа нь бууруулах үед системийн ерөнхий гүйцэтгэлийг ажиглав. Baseline-ийн **p95 = 229.18 ms** үр дүнд үндэслэн тодорхойлсон  **error rate < 1%** гэсэн SLO шаардлагуудыг PASS тестээр хангасан. Харин **p95 < 50 ms** гэсэн зориудаар хатуу тогтоосон threshold хангагдаагүй тул FAIL болсон бөгөөд энэ нь k6-г гүйцэтгэлийн шаардлагыг автоматаар шалгах quality gate болгон ашиглах боломжтойг харуулав.
## Files

- `script.js`
- `script-stages.js`
- `script-threshold.js`
- `results/run-05vu.txt`
- `results/run-30vu.txt`
- `results/run-100vu.txt`
- `results/run-stages.txt`
- `results/run-threshold-pass.txt`
- `results/run-threshold-fail.txt`
- `results/screenshots/05.png`
- `results/screenshots/30.png`
- `results/screenshots/100.png`
- `results/screenshots/fail.png`
- `results/screenshots/succeed.png`
- `results/screenshots/stages.png`