OWASP API05:2023 - Broken Function Level Authorization (BFLA) Lab Walkthrough

Vulnerability Overview

During a targeted API security assessment within the Completely Ridiculous API (crAPI) environment, I identified a high-severity Broken Function Level Authorization (BFLA) vulnerability.

The flaw resides within the video profile management microservice layer. Because the backend implementation relies on structural obscurity to hide administrative operations rather than validating the specific role memberships of incoming session requests, low-privileged users can execute unauthorized administrative functions—specifically, the permanent deletion of arbitrary video assets

Technical Walkthrough & Exploitation Lifecycle

1. Frontend & Resource Mapping
The assessment began on the user dashboard interface, where standard accounts can upload and manage personal video clips attached to their vehicle profiles.
<img width="732" height="392" alt="Screenshot 2026-06-16 040053" src="https://github.com/user-attachments/assets/95707dd0-af15-4029-850e-a917e86a3e8d" />
By interacting with the video player component on the frontend, I generated baseline application traffic to isolate the backend endpoints responsible for handling asset delivery.

2.  Endpoint Analysis
Using Burp Suite Proxy, I audited the chronological HTTP history to trace the exact microservice routing strings used by the web application.
<img width="803" height="387" alt="Screenshot 2026-06-16 040218" src="https://github.com/user-attachments/assets/0b179d54-5935-40f9-8175-8055d45f7548" />
Out of the active stream of traffic, I isolated a stable REST endpoint tracking video resources by object IDs under the identity service layer:
GET /identity/api/v2/user/videos/103 HTTP/1.1

3. Baseline Validation
To verify the standard operational behavior of the API, the captured ⁠GET⁠ transaction was forwarded to Burp Suite Repeater.
<img width="800" height="389" alt="Screenshot 2026-06-16 031529" src="https://github.com/user-attachments/assets/c7cb9b23-38f3-4a2e-a06b-1a8109841b5f" />
The application responded with a standard ⁠200 OK⁠ status code, returning a structured JSON payload containing metadata and the Base64-encoded file contents matching video resource ID ⁠103⁠.

4. Method Fuzzing & Information Leakage
To evaluate the access control restrictions governing this object path, I modified the HTTP transaction verb from a read-only ⁠GET⁠ to a destructive ⁠DELETE⁠ statement
<img width="794" height="397" alt="Screenshot 2026-06-16 031422" src="https://github.com/user-attachments/assets/53c073c4-9d23-483e-877b-c7b8d0302f23" />
DELETE /identity/api/v2/user/videos/103 HTTP/1.1
The endpoint successfully blocked direct execution, returning a ⁠403 Forbidden⁠ response. However, the backend application logic leaked an explicitly descriptive structural error message:

{
  "message": "This is an admin function. Try to access the admin API",
  "status": 403
}
This exception response explicitly confirmed that the controller layer maps deletion capabilities to a parallel administrative URL directory scheme.

5. Executing the BFLA Bypass

Using the structural disclosure provided by the application's error routine, I manually re-mapped the URL directory structure. I swapped the privilege context within the path string from ⁠user⁠ to ⁠admin⁠ while keeping the original low-privileged standard user session token attached to the ⁠Authorization: Bearer⁠ header:
<img width="800" height="392" alt="Screenshot 2026-06-16 031621" src="https://github.com/user-attachments/assets/084b87d7-4a7b-4688-869d-80f0137c1ad5" />
DELETE /identity/api/v2/admin/videos/103 HTTP/1.1
The backend microservice accepted the path modification and processed the command. Because the authorization filter only checked that my session token was active but failed to audit whether my specific identity possessed administrative privileges, the function executed completely.
The API returned a ⁠200 OK⁠ response payload explicitly confirming:
{
  "message": "User video deleted successfully.",
  "status": 200
}

 Remediation
To mitigate this vulnerability and enforce strict isolation between access tiers, the following controls should be applied:
 Server-Side Authorization Checks (RBAC): Never rely on parallel URL directories (⁠/admin/⁠ vs ⁠/user/⁠) to enforce security boundaries. The application must explicitly decode the incoming JSON Web Token (JWT) on the backend and verify that the user's mapped role matches an authorized access control list before handling administrative methods.
 Controller-Level Access Control: Implement explicit validation policies directly inside the resource controllers. If an endpoint receives a ⁠DELETE⁠ request, it must strictly confirm that the session holder's identity matches either the absolute owner of that specific resource ID or a globally verified administrative account.



