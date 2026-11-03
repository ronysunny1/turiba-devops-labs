# Lab 6 · Test what you ship, ship a release

GitHub user: ronysunny1 · Reference pipeline used (Part 0 fallback): no

<!-- Keep every heading as it is: the grader finds the sections by name. Replace every blank (four underscores), and <you> in the first line and in section 3. -->

## 1 · Smoke test

- The names of my smoke-test steps in the `image` job: `Build for the smoke test (amd64, kept on the runner)` and `Smoke test the image`
- What my smoke test asks, and what makes it fail: It starts a container using `docker run -d --name smoke -p 5000:5000 "$IMAGE:${{ github.sha }}"` and queries `GET /` using `curl --silent --show-error --fail --retry 10 --retry-delay 1 --retry-all-errors http://localhost:5000/`. It fails if curl receives a non-2xx HTTP status code or fails to connect after 10 retries, which then prints the container output using `docker logs smoke` and exits with status code 1.
- 🔮 Part 1, before my first run: will my code pass the smoke test the first time? I predicted yes because the API was already tested and working in Lab 5. What happened: The initial run failed due to a YAML indentation error on the new steps, but passed once the workflow indentation was fixed.
- 🔮 Part 2, before the checks finished: which checks will go red after `break.sh`? I predicted image because the teammate's commit added a new dependency that might not be properly included in the Docker build context.

## 2 · What broke and how I found it

- The check and the step that went red: Check `image`, step `Smoke test the image`
- The log line that showed the cause (copied from the log):

```text
Error: Cannot find module './banner'
