# Lab 9 Â· Package it with Helm

GitHub user: ronysunny1 Â· Fallback image used: no Â· Fallback Mission 8 files used: no

<!-- Keep every heading as it is: the grader finds the sections by name. Replace every blank (four underscores). Paste command output inside the ```text blocks. Never paste your password. -->

## 1 Â· The chart

```text
$ I elm lint charts/course-app
==> Linting charts/course-app
[INFO] Chart.yaml: icon is recommended

1 chart(s) linted, 0 chart(s) failed

8 helm template course-app charts/course-app -n course --set api.replicas=2 | grep -E 'replicas:|image:'
  replicas: 2
          image: postgres:17-alpine
          image: "ghcr.io/ronysunny1/turiba-devops-labs/api:0.2.0"
  replicas: 1
          image: postgres:17-alpine
```

- ðŸ”® Before running `helm template â€¦ --set api.replicas=2`: which lines of the output change, and which file decides that? I predicted that the API Deployment's `replicas` line would change from `3`to `2` while the database `replicas` stayed `1`, decided by `charts/course-app/values.yaml` (and referenced in `charts/course-app/templates/api-deployment.yaml`) because `--set api.replicas=2` overrides the default value in `values.yaml`. What happened: the API Deployment rendered `replicas: 2` while the DB Deployment remained `replicas: 1`.
- In my own words, why the Namespace and the Secret are **not** in the chart: The `course` namespace is the target environment boundary that already exists before Helm installs a release into it, and `db-secret` contains sensitive database credentials that must never be hardcoded or committed into a Git repository in plain text.

## 2 Â· Install, upgrade, rollback

```text
$ I elm install course-app charts/course-app -n course
NAME: course-app
LAST DEPLOYED: Mon Nov 16 19:26:01 2026
NAMESPACE: course
STATUS: deployed
REVISION: 1
DESCRIPTION: Install complete
TEST SUITE: None

$ I elm upgrade course-app charts/course-app -n course --set api.image.tag=does-not-exist
Release "course-app" has been upgraded. Happy Helming!
NAME: course-app
LAST DEPLOYED: Mon Nov 16 19:24:37 2026
NAMESPACE: course
STATUS: deployed
REVISION: 3
DESCRIPTION: Upgrade complete
TEST SUITE: None

$ kubectl -n course get pods -l app=api        (about 30 seconds later)
NAME                   READY   STATUS         RESTARTS   AGE
api-5895fc94dc-6szmm   0/1     ErrImagePull   0          30s
api-7cd46cd78d-5pqm2   1/1     Running        0          6m23s
api-7cd46cd78d-cqwrp   1/1     Running        0          6m23s
api-7cd46cd78d-mlvqs   1/1     Running        0          6m23s

$ I elm history course-app -n course            (before the rollback)
REVISION        UPDATE@                        STATUS          CHART               APP VERSION     DESCRIPTION
1                Mon Nov 16 19:18:45 2026        superseded      course-app-0.1.0    0.2.0           Install complete
2               Mon Nov 16 19:23:53 2026        superseded      course-app-0.1.0    0.2.0           Upgrade complete
3               Mon Nov 16 19:24:37 2026        deployed        course-app-0.1.0    0.2.0           Upgrade complete

$ I elm rollback course-app 1 -n course
Rollback was a success! Happy Helming!

$ helm history course-app -n course            (after the rollback)
REVISION        UPDATE@                        STATUS          CHART               APP VERSION     DESCRIPTION
1                Mon Nov 16 19:18:45 2026        superseded      course-app-0.1.0    0.2.0           Install complete
2               Mon Nov 16 19:23:53 2026        superseded      course-app-0.1.0    0.2.0           Upgrade complete
3               Mon Nov 16 19:24:37 2026        superseded      course-app-0.1.0    0.2.0           Upgrade complete
4               Mon Nov 16 19:25:34 2026        deployed        course-app-0.1.0    0.2.0           Rollback to 1
```

-<'å+ˆ™Y›Ü™HH\Ü˜YHÚ]HYÈ]Ù\Û‰Ý^\ÝˆÚ[[H\Ü˜YXØ^H]˜Z[YÈÚ[ÑUØ\KÝÙÜØÝ[[œÝÙ\ÈH™YXÝY][H\Ü˜YXÛÝ[™\ÜÕUTÎˆ\ÞYY˜]\ˆ[ˆ˜Z[[™È[™ÑUØ\KÝÙÜØÛÝ[ÛÛ[YH[œÝÙ\š[™È™XØ]\ÙH[HÛ›HÚXÚÜÈÚ]\ˆÝX™\›™]\ÈXØÙ\YH\]Y\Þ[Y[ÜXÈÚ[HÝX™\›™]\ÉÜÈ›Û[™Õ\]XÝ˜]YÞHÙY\È^\Ý[™ÈX[HÙÈ[›š[™ÈÚ[ˆH™]È™\XØHØ[››Ý[]È[XYÙKˆÚ]\[™Yˆ[H\Ü˜YX™\ÜYÕUTÎˆ\ÞYYH™]ÈÙ˜Z[YÚ]\œ’[XYÙT[Ú[HHÈ^\Ý[™È[›š[™ØÙÈÝ^YY\[™ÑUØ\KÝÙÜØÙ\™]\›š[™ÈHØ]™YÙË‚‹H<'å+ˆ™Y›Ü™HH›Û˜XÚÎˆÚXÚ™]š\Ú[Ûˆ[X™\ˆÚ[[H›Û˜XÚÈÛÝ\œÙKX\XÜ™X]OÈH™YXÝY]]ÛÝ[Ü™X]HH™]È›ÜØ\™™]š\Ú[Ûˆ
™]š\Ú[Ûˆ›ÛÝÚ[™È™]š\Ú[ÛˆÊH˜]\ˆ[ˆ™]Ú[™[™È\ÝÜžH™XØ]\ÙH[H™]™\ˆ[]\ÈÜˆÝ™\Üš]\È™[X\ÙH\ÝÜžH[™[œÝXY\[™È]™\žH›Û˜XÚÈ\ÈH™]È™]š\Ú[ÛˆÚ]H\™Ù]™]š\Ú[Û‰ÜÈÛÛ™šYÝ\˜][Û‹ˆÚ]\[™Yˆ[H›Û˜XÚÈÛÝ\œÙKX\XÜ™X]Y™]š\Ú[Ûˆ
›Û˜XÚÈÈX
H[™™\ÝÜ™Y[È\X™\XØ\ÈÈKÌH[›š[™Ø‚‹H<'å+ˆ™Y›Ü™H[H[š[œÝ[[™H™]È[H[œÝ[ˆÚ[[Ý\ˆÙÈÝ[™H\™OÈH™YXÝY]HÙÈÛÝ[™HÛÛ™HY\ˆ[H[š[œÝ[[™[H[œÝ[™XØ]\ÙHH‹Y]X\œÚ\Ý[›Û[YPÛZ[X\ÈYš[™Y[œÚYHÚ\ËØÛÝ\œÙKX\Ý[\]\ËØÛÈ[H[š[œÝ[[]\ÈHÈ[Û™ÈÚ]]ÈÝÜ™YÜÝÜ™TÔS]KˆÚ]\[™YˆÝ\›\ÈØØ[ÜÝŽØ\KÝÙÜØ™]\›™Y×X
[\H\œ˜^JKÛÛ™š\›Z[™ÈH™]š[Ý\È]HØ\È[]YÚ]HÚ\	ÜÈË‚‚˜^‰Ý\›\ÈØØ[ÜÝŽØ\KÝÙÜÈ
Y\ˆ[š[œÝ[[™[œÝ[
B–×B˜‚ˆÈÈÈ0­ÈÚ]œ›ÚÙH[™ÝÈH›Ý[™]‚‹H<'å+ˆ™Y›Ü™H\Ü˜Y[™ÈÚ]HX[[X]IÜÈœ˜[˜ÚH^XÝY\ÈÈÛÈÜ›Û™Èš\œÝˆH^XÝY[H\Ü˜YXÈ˜Z[ÛˆHÛÛ™›XÝÚ]H™]ÛHYY[\]\ËÛ˜[Y\ÜXÙKžX[[™\ÛÝ\˜ÙH™XØ]\ÙH˜[Y\ÜXÙKØÛÝ\œÙX[™XYH^\ÝY[ˆHÛ\Ý\ˆÚ]Ý][HÝÛ™\œÚ\Y]Y]K‚‚ˆÈÚ]HØ]ÈHÛÛ[X[™[™H[™HÙˆÝ]]]ÚÝÙY]^Hš^
ÛÛ[Z]
HŸKNŸKKKKKKKKKKK_KKKKKKKKKKKKKKKKKKKKKKKKKKKKKKKKKKKKKKKKKKKKKKKKKKKK_KKKKKKKKKKKKKKKK_ŸH[\]Hš[Ú[\ˆ\œ›Üˆ™XØ]\ÙH\KY\Þ[Y[žX[[™Y™\™[˜Ù