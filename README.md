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
[энд k6 version командын бүтэн output-ийг оруул]
```

- Operating system: `[Ubuntu / macOS / WSL2 гэх мэт]`
- Test date: `[2026-09-14 эсвэл бодит огноо]`
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

## VU comparison

| VU | p90 latency | p95 latency | Throughput | Error rate |
|---:|---:|---:|---:|---:|
| 5 | 230.55 ms | 272.32 ms | 7.40 req/s | 0.00% |
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

Stages тестээр ачаалал аажмаар 5 → 30 → 100 VU болж өсөж, дараа нь 0 болж буурсан ерөнхий хэлбэрийг ажиглав. Нийт 7072 request, дундаж throughput 46.87 req/s, p95 latency 231.15 ms, error rate 0.00% байсан бөгөөд гурван түвшний тоон харьцуулалтыг stages-ийн нийлбэр summary-ээс бус, тусдаа 5, 30, 100 VU утгуудаар авав.

## SLO and thresholds

5 VU-ийн baseline p95 latency нь 272.32 ms байсан. Үүн дээр үндэслэн p95 latency-ийн SLO-г 408 ms-ээс бага байхаар сонгосон. Энэ босго нь baseline-оос тодорхой хэмжээний хэлбэлзлийг зөвшөөрөх боловч хэрэглэгчийн хүлээлтийн хувьд хэт удаан хариуг илрүүлэх зорилготой.

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

Санаатайгаар хатуу threshold ашигласан FAIL үр дүн:

```text
results/run-threshold-fail.txt
```

PASS тестэд `p(95)<408` болон `rate<0.01` threshold хоёулаа хангагдсан (p95 = 233.58 ms). FAIL тестэд `p(95)<50` threshold хангагдаагүй (p95 = 232.43 ms) тул k6 тестийг амжилтгүй гэж тэмдэглэв.

## Дүгнэлт

5 VU-ийн үед системийн p95 latency 272.32 ms байсан бөгөөд энэ нь baseline гүйцэтгэлийг харьцуулж өгсөг байгаа. Харин 30 VU хүртэл ачаалал нэмэгдэхэд throughput 7.40 req/s-ээс 45.47 req/s болж өсөв. Үүний зэрэгцээ p95 latency 272.32 ms-ээс 231.38 ms болж буурсан. 100 VU-ийн үед нэгж хэрэглэгчийн хариу хүлээх хугацаа 232.14 ms байж, 30 VU-тэй харьцуулахад бага зэрэг өссөн ч ерөнхийдөө тогтвортой хэвээр байсан. Энэ нь зэрэгцээ хэрэглэгчийн тоо өсөхөд системийн нөөц болон сүлжээний нөхцөл гүйцэтгэлд нөлөөлдгийг харуулсан. Хэрэглэгчийн туршлага 100 VU орчимд бага зэрэг муудаж эхэлсэн ч нийтдээ тогтвортой. Stages тестээр ачаалал өсөх, оргилд хүрэх, буурах үеийн ерөнхий хэлбэрийг ажигласан. Threshold тест нь SLO-г зөвхөн хэмжих биш, автоматаар PASS эсвэл FAIL болгон шалгаж болдгийг харуулсан. `p(95)<408` болон error rate `<1%` гэсэн SLO нь PASS тестэд хангагдсан. Харин хэт хатуу threshold ашиглахад quality gate FAIL болсныг тусдаа гаралтаар батлан харуулав.

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
- Screenshot files