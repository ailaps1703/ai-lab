# ai-lab

Here’s a concise product diagram description plus a phase-by-phase checklist you can directly implement.

## Product / Architecture Diagram (what to visualize)

Draw a left-to-right flow:

- Viewer (Browser)
  - Label: “User / Viewer”
  - Connect via “HTTPS (public)” arrow to…

- CloudFront + AWS WAF
  - Box label: “Amazon CloudFront (Viewer URL: d1234abcd.cloudfront.net) + AWS WAF (L1)”
  - Inside annotations:
    - Viewer Protocol Policy: Redirect HTTP → HTTPS
    - WebSocket supported
    - Origin Custom Header: X-Origin-Verify: <secret>
  - Arrow “HTTPS :443” to…

- EC2 Instance (Origin) in Customer VPC
  - Outer box label: “Customer AWS Account – VPC – EC2 Origin”
  - Inside, draw:
    - Nginx Container
      - Label: “Nginx (TLS termination, L3: X-Origin-Verify, 443)”  
      - Connected via HTTP arrow to:
    - OpenClaw Gateway Container
      - Label: “OpenClaw Gateway (HTTP 127.0.0.1:18789 in Docker bridge, L4: token auth)”
    - Certbot Container
      - Label: “Certbot (Let’s Encrypt DNS-01 via SaaS Route53)”
  - Attach to EC2:
    - “EC2 Security Group (L2: allow 443 only from CloudFront managed prefix list)”
    - “IAM Instance Role -> Secrets Manager, SSM, STS to SaaS DNS role”

- SaaS Platform (Control Plane)
  - Separate box: “OpenClaw SaaS Backend (SaaS AWS Account)”
  - Arrows/notes:
    - Uses cross-account AssumeRole (with ExternalId) into customer account
    - Creates VPC, SG, EC2, Secrets, WAF, CloudFront, Route53 origin record
    - Manages:
      - Origin domain: <tenantSlug>-origin.openclaw-saas.io
      - Let’s Encrypt DNS-01 on SaaS Route53 zone
      - X-Origin-Verify secret lifecycle
      - Gateway token secret

- Security Layers (side annotation)
  - L1: CloudFront + WAF – “Is this traffic safe?”
  - L2: SG – “Is this from CloudFront IPs?”
  - L3: Nginx header – “Is this from our CloudFront distribution?”
  - L4: Gateway token – “Is this user authorized to use OpenClaw?”

Also show the two key DNS names:

- Public entry point: d1234abcd.cloudfront.net (customer later CNAMEs their own domain here).
- Internal origin: <tenantSlug>-origin.openclaw-saas.io (used only by CloudFront, never by users). [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/21376642/a796a81a-f761-4d35-9b61-3d23e56c3ccf/Variant-A-Dockerized-Implementation-Only-CF.md.pdf?AWSAccessKeyId=ASIA2F3EMEYEW4EPXNFY&Signature=pAvf3Las94AdINcb69y0OoRenvA%3D&x-amz-security-token=IQoJb3JpZ2luX2VjEHYaCXVzLWVhc3QtMSJGMEQCIAiceeSAKm2Evm%2FTghzxLm7jtEMfqq9Irp2Vq%2B2nv7iVAiBrK22D6HLaCgYeqp%2B1g%2FQCp7y2ngekYn4gm6dhDj7KbSrzBAg%2FEAEaDDY5OTc1MzMwOTcwNSIMmsH3WZs8ZkaGyt%2BmKtAErzRg%2FqJcyd2IMxB3whwIHWwfHpCYj5xyFSx3P7ZYk7qExPqmuSqNL%2FIHEh4u0ZYObA9MexoY3JVwcdFC0ffFNoMvvSc917fcSnt5jZjU6fDLhiwBrj88OJ7357q4SW%2FH1dX1%2FXYJHKCrfQUjNUU230OQt9OWyh6aCQzzBNKoC%2BeXMj4FYEHqqPAgbwnQuC7MeGI%2Fk2Xrb4Fob2Uoo%2FZwzzVM00EuMIQyajQL5KKcmDHNxA4D4lBX4XwZNa4uJV%2B18r1wPOfMnqyQJ%2BseaCGktpXcYDjB3IhiEJCYCLYLit75FYK04LjJfrSwi8QBuCUtS0BimHtWsxMOJENsbK8rY6Li22EQu%2F92aUdqnr2CtylIZ3qolQUMHaVkyZ4zOzHZx4dIGwPyO7wrKfhxJw9aBOl28FNhfDxABmBvMYDTikeIfssJMnZ5L%2FTLmCuX2DSyZ69LAZCpIpxG8gyez0O6%2F%2BBbC9lbb3TSKE4i5TcKjV%2FXrUIWPyoe4IT9zzd4%2BGYRiec2FX7J6mhYb046Kl2LWWEAZr0Awx%2Fr2dkCZcqya8VfVh1VgdAcidVLzXEzD9Ff8wf2y6p3d5Q%2FMnEMqfUkxc0Hw96hyLvDBOXWcm8xFy%2Bpn1Xi90ggqNmMceKqHs69bVYFFynprJiFtzum3IjOFjBpwQEGi7PJBrb22ux3eIfRAi50w86Djrx%2Fsj%2FpOTbpfNuDl2sg6oxmHbjzSEt9f6WGL0Tdld0Nhgd%2FZspKTqjKcdRwsJ0kpisj2XpzZTur2sRMyaf7Lrcgy1ZKm3CfYzDA4r7NBjqZAVxaI87IMhVLOtEzw9hVJ3NlFpdcV9yk1GGV9GUxJtAwxS%2F0m9PltdiaU6BNn%2BlsZQzuM9SBYOoB80kEQ5A6lBXgvQorXcmDQRjSXKWQxAgthWeqNAH%2BG6iTsblWm9avUTFRmzMH8gHrT9aLi%2FdSUh7js7gvO2Z7dVWZP%2F%2FQi3x%2BaJM1mNLx%2BSJ3KHL6kjgmeBljMffxiK1i0g%3D%3D&Expires=1773123097)

***

## Step‑by‑step bullets (implementation checklist)

I’ve preserved the phase structure but compressed into actionable bullets you can follow when implementing the automation.

### Phase 0 – Governance & Preconditions

- Generate a high‑entropy `openclaw_gateway_token` in SaaS backend.
- Assume customer cross‑account role via STS (with ExternalId).
- Verify SaaS Route53 hosted zone for `openclaw-saas.io` exists and is authoritative.
- Confirm no root creds or long‑lived IAM keys are stored in SaaS; only role metadata. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/21376642/a796a81a-f761-4d35-9b61-3d23e56c3ccf/Variant-A-Dockerized-Implementation-Only-CF.md.pdf?AWSAccessKeyId=ASIA2F3EMEYEW4EPXNFY&Signature=pAvf3Las94AdINcb69y0OoRenvA%3D&x-amz-security-token=IQoJb3JpZ2luX2VjEHYaCXVzLWVhc3QtMSJGMEQCIAiceeSAKm2Evm%2FTghzxLm7jtEMfqq9Irp2Vq%2B2nv7iVAiBrK22D6HLaCgYeqp%2B1g%2FQCp7y2ngekYn4gm6dhDj7KbSrzBAg%2FEAEaDDY5OTc1MzMwOTcwNSIMmsH3WZs8ZkaGyt%2BmKtAErzRg%2FqJcyd2IMxB3whwIHWwfHpCYj5xyFSx3P7ZYk7qExPqmuSqNL%2FIHEh4u0ZYObA9MexoY3JVwcdFC0ffFNoMvvSc917fcSnt5jZjU6fDLhiwBrj88OJ7357q4SW%2FH1dX1%2FXYJHKCrfQUjNUU230OQt9OWyh6aCQzzBNKoC%2BeXMj4FYEHqqPAgbwnQuC7MeGI%2Fk2Xrb4Fob2Uoo%2FZwzzVM00EuMIQyajQL5KKcmDHNxA4D4lBX4XwZNa4uJV%2B18r1wPOfMnqyQJ%2BseaCGktpXcYDjB3IhiEJCYCLYLit75FYK04LjJfrSwi8QBuCUtS0BimHtWsxMOJENsbK8rY6Li22EQu%2F92aUdqnr2CtylIZ3qolQUMHaVkyZ4zOzHZx4dIGwPyO7wrKfhxJw9aBOl28FNhfDxABmBvMYDTikeIfssJMnZ5L%2FTLmCuX2DSyZ69LAZCpIpxG8gyez0O6%2F%2BBbC9lbb3TSKE4i5TcKjV%2FXrUIWPyoe4IT9zzd4%2BGYRiec2FX7J6mhYb046Kl2LWWEAZr0Awx%2Fr2dkCZcqya8VfVh1VgdAcidVLzXEzD9Ff8wf2y6p3d5Q%2FMnEMqfUkxc0Hw96hyLvDBOXWcm8xFy%2Bpn1Xi90ggqNmMceKqHs69bVYFFynprJiFtzum3IjOFjBpwQEGi7PJBrb22ux3eIfRAi50w86Djrx%2Fsj%2FpOTbpfNuDl2sg6oxmHbjzSEt9f6WGL0Tdld0Nhgd%2FZspKTqjKcdRwsJ0kpisj2XpzZTur2sRMyaf7Lrcgy1ZKm3CfYzDA4r7NBjqZAVxaI87IMhVLOtEzw9hVJ3NlFpdcV9yk1GGV9GUxJtAwxS%2F0m9PltdiaU6BNn%2BlsZQzuM9SBYOoB80kEQ5A6lBXgvQorXcmDQRjSXKWQxAgthWeqNAH%2BG6iTsblWm9avUTFRmzMH8gHrT9aLi%2FdSUh7js7gvO2Z7dVWZP%2F%2FQi3x%2BaJM1mNLx%2BSJ3KHL6kjgmeBljMffxiK1i0g%3D%3D&Expires=1773123097)

### Phase 1 – SST Project / IaC Skeleton

- Define environment stages: dev, qa, prod – each fully isolated.
- Create a Zod schema for all customer inputs (accountId, role ARN, region, tenantSlug, etc.).
- Standardize naming: `openclaw-<stage>-<resource>` for all AWS resources.
- Define outputs contract: distribution URL, origin IP, secret ARNs, WAF ACL ARN, etc. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/21376642/a796a81a-f761-4d35-9b61-3d23e56c3ccf/Variant-A-Dockerized-Implementation-Only-CF.md.pdf?AWSAccessKeyId=ASIA2F3EMEYEW4EPXNFY&Signature=pAvf3Las94AdINcb69y0OoRenvA%3D&x-amz-security-token=IQoJb3JpZ2luX2VjEHYaCXVzLWVhc3QtMSJGMEQCIAiceeSAKm2Evm%2FTghzxLm7jtEMfqq9Irp2Vq%2B2nv7iVAiBrK22D6HLaCgYeqp%2B1g%2FQCp7y2ngekYn4gm6dhDj7KbSrzBAg%2FEAEaDDY5OTc1MzMwOTcwNSIMmsH3WZs8ZkaGyt%2BmKtAErzRg%2FqJcyd2IMxB3whwIHWwfHpCYj5xyFSx3P7ZYk7qExPqmuSqNL%2FIHEh4u0ZYObA9MexoY3JVwcdFC0ffFNoMvvSc917fcSnt5jZjU6fDLhiwBrj88OJ7357q4SW%2FH1dX1%2FXYJHKCrfQUjNUU230OQt9OWyh6aCQzzBNKoC%2BeXMj4FYEHqqPAgbwnQuC7MeGI%2Fk2Xrb4Fob2Uoo%2FZwzzVM00EuMIQyajQL5KKcmDHNxA4D4lBX4XwZNa4uJV%2B18r1wPOfMnqyQJ%2BseaCGktpXcYDjB3IhiEJCYCLYLit75FYK04LjJfrSwi8QBuCUtS0BimHtWsxMOJENsbK8rY6Li22EQu%2F92aUdqnr2CtylIZ3qolQUMHaVkyZ4zOzHZx4dIGwPyO7wrKfhxJw9aBOl28FNhfDxABmBvMYDTikeIfssJMnZ5L%2FTLmCuX2DSyZ69LAZCpIpxG8gyez0O6%2F%2BBbC9lbb3TSKE4i5TcKjV%2FXrUIWPyoe4IT9zzd4%2BGYRiec2FX7J6mhYb046Kl2LWWEAZr0Awx%2Fr2dkCZcqya8VfVh1VgdAcidVLzXEzD9Ff8wf2y6p3d5Q%2FMnEMqfUkxc0Hw96hyLvDBOXWcm8xFy%2Bpn1Xi90ggqNmMceKqHs69bVYFFynprJiFtzum3IjOFjBpwQEGi7PJBrb22ux3eIfRAi50w86Djrx%2Fsj%2FpOTbpfNuDl2sg6oxmHbjzSEt9f6WGL0Tdld0Nhgd%2FZspKTqjKcdRwsJ0kpisj2XpzZTur2sRMyaf7Lrcgy1ZKm3CfYzDA4r7NBjqZAVxaI87IMhVLOtEzw9hVJ3NlFpdcV9yk1GGV9GUxJtAwxS%2F0m9PltdiaU6BNn%2BlsZQzuM9SBYOoB80kEQ5A6lBXgvQorXcmDQRjSXKWQxAgthWeqNAH%2BG6iTsblWm9avUTFRmzMH8gHrT9aLi%2FdSUh7js7gvO2Z7dVWZP%2F%2FQi3x%2BaJM1mNLx%2BSJ3KHL6kjgmeBljMffxiK1i0g%3D%3D&Expires=1773123097)

### Phase 2 – EC2 Instance Role (runtime identity)

- On each deploy phase, call `sts:AssumeRole` into customer account with ExternalId.
- Create an EC2 IAM role + instance profile with least privilege:
  - `secretsmanager:GetSecretValue` scoped to `openclaw-<stage>-*`.
  - `sts:AssumeRole` to SaaS DNS role (for Certbot Route53).
  - SSM: `UpdateInstanceInformation`, `StartSession`, `SendCommand`.
- Attach instance profile to the future EC2 instance. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/21376642/a796a81a-f761-4d35-9b61-3d23e56c3ccf/Variant-A-Dockerized-Implementation-Only-CF.md.pdf?AWSAccessKeyId=ASIA2F3EMEYEW4EPXNFY&Signature=pAvf3Las94AdINcb69y0OoRenvA%3D&x-amz-security-token=IQoJb3JpZ2luX2VjEHYaCXVzLWVhc3QtMSJGMEQCIAiceeSAKm2Evm%2FTghzxLm7jtEMfqq9Irp2Vq%2B2nv7iVAiBrK22D6HLaCgYeqp%2B1g%2FQCp7y2ngekYn4gm6dhDj7KbSrzBAg%2FEAEaDDY5OTc1MzMwOTcwNSIMmsH3WZs8ZkaGyt%2BmKtAErzRg%2FqJcyd2IMxB3whwIHWwfHpCYj5xyFSx3P7ZYk7qExPqmuSqNL%2FIHEh4u0ZYObA9MexoY3JVwcdFC0ffFNoMvvSc917fcSnt5jZjU6fDLhiwBrj88OJ7357q4SW%2FH1dX1%2FXYJHKCrfQUjNUU230OQt9OWyh6aCQzzBNKoC%2BeXMj4FYEHqqPAgbwnQuC7MeGI%2Fk2Xrb4Fob2Uoo%2FZwzzVM00EuMIQyajQL5KKcmDHNxA4D4lBX4XwZNa4uJV%2B18r1wPOfMnqyQJ%2BseaCGktpXcYDjB3IhiEJCYCLYLit75FYK04LjJfrSwi8QBuCUtS0BimHtWsxMOJENsbK8rY6Li22EQu%2F92aUdqnr2CtylIZ3qolQUMHaVkyZ4zOzHZx4dIGwPyO7wrKfhxJw9aBOl28FNhfDxABmBvMYDTikeIfssJMnZ5L%2FTLmCuX2DSyZ69LAZCpIpxG8gyez0O6%2F%2BBbC9lbb3TSKE4i5TcKjV%2FXrUIWPyoe4IT9zzd4%2BGYRiec2FX7J6mhYb046Kl2LWWEAZr0Awx%2Fr2dkCZcqya8VfVh1VgdAcidVLzXEzD9Ff8wf2y6p3d5Q%2FMnEMqfUkxc0Hw96hyLvDBOXWcm8xFy%2Bpn1Xi90ggqNmMceKqHs69bVYFFynprJiFtzum3IjOFjBpwQEGi7PJBrb22ux3eIfRAi50w86Djrx%2Fsj%2FpOTbpfNuDl2sg6oxmHbjzSEt9f6WGL0Tdld0Nhgd%2FZspKTqjKcdRwsJ0kpisj2XpzZTur2sRMyaf7Lrcgy1ZKm3CfYzDA4r7NBjqZAVxaI87IMhVLOtEzw9hVJ3NlFpdcV9yk1GGV9GUxJtAwxS%2F0m9PltdiaU6BNn%2BlsZQzuM9SBYOoB80kEQ5A6lBXgvQorXcmDQRjSXKWQxAgthWeqNAH%2BG6iTsblWm9avUTFRmzMH8gHrT9aLi%2FdSUh7js7gvO2Z7dVWZP%2F%2FQi3x%2BaJM1mNLx%2BSJ3KHL6kjgmeBljMffxiK1i0g%3D%3D&Expires=1773123097)

### Phase 3 – Network & Origin Lockdown

- Create VPC security group for origin.
- Add inbound rule:
  - 443/tcp from CloudFront managed prefix list `com.amazonaws.global.cloudfront.origin-facing` when `restrictOriginToCloudFront = true`.
- Optionally add 22/tcp from `sshAllowedCidrs` when `enableSsh = true`.
- Do not expose port 18789 anywhere; OpenClaw stays internal on Docker bridge. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/21376642/a796a81a-f761-4d35-9b61-3d23e56c3ccf/Variant-A-Dockerized-Implementation-Only-CF.md.pdf?AWSAccessKeyId=ASIA2F3EMEYEW4EPXNFY&Signature=pAvf3Las94AdINcb69y0OoRenvA%3D&x-amz-security-token=IQoJb3JpZ2luX2VjEHYaCXVzLWVhc3QtMSJGMEQCIAiceeSAKm2Evm%2FTghzxLm7jtEMfqq9Irp2Vq%2B2nv7iVAiBrK22D6HLaCgYeqp%2B1g%2FQCp7y2ngekYn4gm6dhDj7KbSrzBAg%2FEAEaDDY5OTc1MzMwOTcwNSIMmsH3WZs8ZkaGyt%2BmKtAErzRg%2FqJcyd2IMxB3whwIHWwfHpCYj5xyFSx3P7ZYk7qExPqmuSqNL%2FIHEh4u0ZYObA9MexoY3JVwcdFC0ffFNoMvvSc917fcSnt5jZjU6fDLhiwBrj88OJ7357q4SW%2FH1dX1%2FXYJHKCrfQUjNUU230OQt9OWyh6aCQzzBNKoC%2BeXMj4FYEHqqPAgbwnQuC7MeGI%2Fk2Xrb4Fob2Uoo%2FZwzzVM00EuMIQyajQL5KKcmDHNxA4D4lBX4XwZNa4uJV%2B18r1wPOfMnqyQJ%2BseaCGktpXcYDjB3IhiEJCYCLYLit75FYK04LjJfrSwi8QBuCUtS0BimHtWsxMOJENsbK8rY6Li22EQu%2F92aUdqnr2CtylIZ3qolQUMHaVkyZ4zOzHZx4dIGwPyO7wrKfhxJw9aBOl28FNhfDxABmBvMYDTikeIfssJMnZ5L%2FTLmCuX2DSyZ69LAZCpIpxG8gyez0O6%2F%2BBbC9lbb3TSKE4i5TcKjV%2FXrUIWPyoe4IT9zzd4%2BGYRiec2FX7J6mhYb046Kl2LWWEAZr0Awx%2Fr2dkCZcqya8VfVh1VgdAcidVLzXEzD9Ff8wf2y6p3d5Q%2FMnEMqfUkxc0Hw96hyLvDBOXWcm8xFy%2Bpn1Xi90ggqNmMceKqHs69bVYFFynprJiFtzum3IjOFjBpwQEGi7PJBrb22ux3eIfRAi50w86Djrx%2Fsj%2FpOTbpfNuDl2sg6oxmHbjzSEt9f6WGL0Tdld0Nhgd%2FZspKTqjKcdRwsJ0kpisj2XpzZTur2sRMyaf7Lrcgy1ZKm3CfYzDA4r7NBjqZAVxaI87IMhVLOtEzw9hVJ3NlFpdcV9yk1GGV9GUxJtAwxS%2F0m9PltdiaU6BNn%2BlsZQzuM9SBYOoB80kEQ5A6lBXgvQorXcmDQRjSXKWQxAgthWeqNAH%2BG6iTsblWm9avUTFRmzMH8gHrT9aLi%2FdSUh7js7gvO2Z7dVWZP%2F%2FQi3x%2BaJM1mNLx%2BSJ3KHL6kjgmeBljMffxiK1i0g%3D%3D&Expires=1773123097)

### Phase 4 – Secret Provisioning

- Generate:
  - Gateway token (64+ chars, cryptographically random).
  - Origin verify secret (high‑entropy random string) for `X-Origin-Verify`.
- Store in Secrets Manager:
  - `openclaw-<stage>-gateway-token`.
  - `openclaw-<stage>-origin-secret`.
- Tag secrets with `stage`, `tenantId`, `managedBy` for governance.
- Keep gateway token and origin secret fully separate and independently rotatable. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/21376642/a796a81a-f761-4d35-9b61-3d23e56c3ccf/Variant-A-Dockerized-Implementation-Only-CF.md.pdf?AWSAccessKeyId=ASIA2F3EMEYEW4EPXNFY&Signature=pAvf3Las94AdINcb69y0OoRenvA%3D&x-amz-security-token=IQoJb3JpZ2luX2VjEHYaCXVzLWVhc3QtMSJGMEQCIAiceeSAKm2Evm%2FTghzxLm7jtEMfqq9Irp2Vq%2B2nv7iVAiBrK22D6HLaCgYeqp%2B1g%2FQCp7y2ngekYn4gm6dhDj7KbSrzBAg%2FEAEaDDY5OTc1MzMwOTcwNSIMmsH3WZs8ZkaGyt%2BmKtAErzRg%2FqJcyd2IMxB3whwIHWwfHpCYj5xyFSx3P7ZYk7qExPqmuSqNL%2FIHEh4u0ZYObA9MexoY3JVwcdFC0ffFNoMvvSc917fcSnt5jZjU6fDLhiwBrj88OJ7357q4SW%2FH1dX1%2FXYJHKCrfQUjNUU230OQt9OWyh6aCQzzBNKoC%2BeXMj4FYEHqqPAgbwnQuC7MeGI%2Fk2Xrb4Fob2Uoo%2FZwzzVM00EuMIQyajQL5KKcmDHNxA4D4lBX4XwZNa4uJV%2B18r1wPOfMnqyQJ%2BseaCGktpXcYDjB3IhiEJCYCLYLit75FYK04LjJfrSwi8QBuCUtS0BimHtWsxMOJENsbK8rY6Li22EQu%2F92aUdqnr2CtylIZ3qolQUMHaVkyZ4zOzHZx4dIGwPyO7wrKfhxJw9aBOl28FNhfDxABmBvMYDTikeIfssJMnZ5L%2FTLmCuX2DSyZ69LAZCpIpxG8gyez0O6%2F%2BBbC9lbb3TSKE4i5TcKjV%2FXrUIWPyoe4IT9zzd4%2BGYRiec2FX7J6mhYb046Kl2LWWEAZr0Awx%2Fr2dkCZcqya8VfVh1VgdAcidVLzXEzD9Ff8wf2y6p3d5Q%2FMnEMqfUkxc0Hw96hyLvDBOXWcm8xFy%2Bpn1Xi90ggqNmMceKqHs69bVYFFynprJiFtzum3IjOFjBpwQEGi7PJBrb22ux3eIfRAi50w86Djrx%2Fsj%2FpOTbpfNuDl2sg6oxmHbjzSEt9f6WGL0Tdld0Nhgd%2FZspKTqjKcdRwsJ0kpisj2XpzZTur2sRMyaf7Lrcgy1ZKm3CfYzDA4r7NBjqZAVxaI87IMhVLOtEzw9hVJ3NlFpdcV9yk1GGV9GUxJtAwxS%2F0m9PltdiaU6BNn%2BlsZQzuM9SBYOoB80kEQ5A6lBXgvQorXcmDQRjSXKWQxAgthWeqNAH%2BG6iTsblWm9avUTFRmzMH8gHrT9aLi%2FdSUh7js7gvO2Z7dVWZP%2F%2FQi3x%2BaJM1mNLx%2BSJ3KHL6kjgmeBljMffxiK1i0g%3D%3D&Expires=1773123097)

### Phase 5 – EC2 Provisioning & Dockerized Bootstrap

- Launch an Ubuntu EC2 instance:
  - Type at least `t3.small` (2 GiB) – `t3.medium` recommended.
  - Attach VPC / SG from Phase 3 and instance profile from Phase 2.
  - Enable EC2 Auto‑Recovery via CloudWatch alarm.
- Enforce IMDSv2 (HttpTokens: required) and encrypt root EBS volume.
- User‑data/bootstrap script must:
  - Update and install base packages: `curl`, `jq`, `openssl`, `ca-certificates`, `gnupg`.
  - Install Docker Engine + docker‑compose plugin from Docker’s apt repo.
  - Configure Docker `daemon.json` log driver (`json-file`, `max-size`, `max-file`) before any containers run.
  - Retrieve secrets from Secrets Manager into shell variables:
    - `GATEWAY_TOKEN` and `ORIGIN_SECRET`.
  - Prepare directories:
    - `/opt/openclaw`, `/opt/openclaw/nginx`, `/opt/openclaw/certbot/{conf,www}`, `/home/ubuntu/openclaw-data`.
  - Create `docker-compose.yaml` with three services:
    - `openclaw-gateway` container, bound to `0.0.0.0:18789` inside bridge, env token, healthcheck.
    - `nginx` container exposing `443:443`, mounting nginx config and cert directories, depends_on gateway.
    - `certbot` container for DNS‑01.
  - Write `/opt/openclaw/.env` with `OPENCLAW_VERSION` and `GATEWAY_TOKEN` (chmod 600).
  - `docker compose up -d openclaw-gateway`.
  - Write `nginx.conf`:
    - `server_name <tenantSlug>-origin.openclaw-saas.io`.
    - TLS with Let’s Encrypt cert paths.
    - `if ($http_x_origin_verify != '<ORIGIN_SECRET_PLACEHOLDER>') { return 403; }`.
    - Security headers (HSTS, X-Frame-Options, etc.).
    - WebSocket proxy settings to `openclaw-gateway:18789`.
  - Replace placeholder with actual `ORIGIN_SECRET` via `sed`.
  - Run Certbot in Docker: `certonly --dns-route53 -d <tenantSlug>-origin.openclaw-saas.io ...`.
  - `docker compose up -d nginx`.
  - Create cron in `/etc/cron.d/certbot-renew` to run `certbot renew` twice daily with deploy‑hook `nginx -s reload`. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/21376642/a796a81a-f761-4d35-9b61-3d23e56c3ccf/Variant-A-Dockerized-Implementation-Only-CF.md.pdf?AWSAccessKeyId=ASIA2F3EMEYEW4EPXNFY&Signature=pAvf3Las94AdINcb69y0OoRenvA%3D&x-amz-security-token=IQoJb3JpZ2luX2VjEHYaCXVzLWVhc3QtMSJGMEQCIAiceeSAKm2Evm%2FTghzxLm7jtEMfqq9Irp2Vq%2B2nv7iVAiBrK22D6HLaCgYeqp%2B1g%2FQCp7y2ngekYn4gm6dhDj7KbSrzBAg%2FEAEaDDY5OTc1MzMwOTcwNSIMmsH3WZs8ZkaGyt%2BmKtAErzRg%2FqJcyd2IMxB3whwIHWwfHpCYj5xyFSx3P7ZYk7qExPqmuSqNL%2FIHEh4u0ZYObA9MexoY3JVwcdFC0ffFNoMvvSc917fcSnt5jZjU6fDLhiwBrj88OJ7357q4SW%2FH1dX1%2FXYJHKCrfQUjNUU230OQt9OWyh6aCQzzBNKoC%2BeXMj4FYEHqqPAgbwnQuC7MeGI%2Fk2Xrb4Fob2Uoo%2FZwzzVM00EuMIQyajQL5KKcmDHNxA4D4lBX4XwZNa4uJV%2B18r1wPOfMnqyQJ%2BseaCGktpXcYDjB3IhiEJCYCLYLit75FYK04LjJfrSwi8QBuCUtS0BimHtWsxMOJENsbK8rY6Li22EQu%2F92aUdqnr2CtylIZ3qolQUMHaVkyZ4zOzHZx4dIGwPyO7wrKfhxJw9aBOl28FNhfDxABmBvMYDTikeIfssJMnZ5L%2FTLmCuX2DSyZ69LAZCpIpxG8gyez0O6%2F%2BBbC9lbb3TSKE4i5TcKjV%2FXrUIWPyoe4IT9zzd4%2BGYRiec2FX7J6mhYb046Kl2LWWEAZr0Awx%2Fr2dkCZcqya8VfVh1VgdAcidVLzXEzD9Ff8wf2y6p3d5Q%2FMnEMqfUkxc0Hw96hyLvDBOXWcm8xFy%2Bpn1Xi90ggqNmMceKqHs69bVYFFynprJiFtzum3IjOFjBpwQEGi7PJBrb22ux3eIfRAi50w86Djrx%2Fsj%2FpOTbpfNuDl2sg6oxmHbjzSEt9f6WGL0Tdld0Nhgd%2FZspKTqjKcdRwsJ0kpisj2XpzZTur2sRMyaf7Lrcgy1ZKm3CfYzDA4r7NBjqZAVxaI87IMhVLOtEzw9hVJ3NlFpdcV9yk1GGV9GUxJtAwxS%2F0m9PltdiaU6BNn%2BlsZQzuM9SBYOoB80kEQ5A6lBXgvQorXcmDQRjSXKWQxAgthWeqNAH%2BG6iTsblWm9avUTFRmzMH8gHrT9aLi%2FdSUh7js7gvO2Z7dVWZP%2F%2FQi3x%2BaJM1mNLx%2BSJ3KHL6kjgmeBljMffxiK1i0g%3D%3D&Expires=1773123097)

### Phase 6 – OpenClaw Runtime Configuration

- Configure gateway (e.g., `openclaw.json` or env) with:
  - `gateway.mode = "local"`.
  - `gateway.port = 18789`.
  - `gateway.bind = "0.0.0.0"` (container‑local; still not public).
  - `gateway.auth.mode = "token"` with `token` read from `OPENCLAW_GATEWAY_TOKEN` env.
  - `controlUi.allowedOrigins = ["https://<cloudfront-domain>.cloudfront.net"]`.
  - HSTS `max-age` initially low (e.g., 300), to be increased after validation.
- Ensure at startup OpenClaw resolves the token from env and never writes plaintext into persistent config or IaC state. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/21376642/a796a81a-f761-4d35-9b61-3d23e56c3ccf/Variant-A-Dockerized-Implementation-Only-CF.md.pdf?AWSAccessKeyId=ASIA2F3EMEYEW4EPXNFY&Signature=pAvf3Las94AdINcb69y0OoRenvA%3D&x-amz-security-token=IQoJb3JpZ2luX2VjEHYaCXVzLWVhc3QtMSJGMEQCIAiceeSAKm2Evm%2FTghzxLm7jtEMfqq9Irp2Vq%2B2nv7iVAiBrK22D6HLaCgYeqp%2B1g%2FQCp7y2ngekYn4gm6dhDj7KbSrzBAg%2FEAEaDDY5OTc1MzMwOTcwNSIMmsH3WZs8ZkaGyt%2BmKtAErzRg%2FqJcyd2IMxB3whwIHWwfHpCYj5xyFSx3P7ZYk7qExPqmuSqNL%2FIHEh4u0ZYObA9MexoY3JVwcdFC0ffFNoMvvSc917fcSnt5jZjU6fDLhiwBrj88OJ7357q4SW%2FH1dX1%2FXYJHKCrfQUjNUU230OQt9OWyh6aCQzzBNKoC%2BeXMj4FYEHqqPAgbwnQuC7MeGI%2Fk2Xrb4Fob2Uoo%2FZwzzVM00EuMIQyajQL5KKcmDHNxA4D4lBX4XwZNa4uJV%2B18r1wPOfMnqyQJ%2BseaCGktpXcYDjB3IhiEJCYCLYLit75FYK04LjJfrSwi8QBuCUtS0BimHtWsxMOJENsbK8rY6Li22EQu%2F92aUdqnr2CtylIZ3qolQUMHaVkyZ4zOzHZx4dIGwPyO7wrKfhxJw9aBOl28FNhfDxABmBvMYDTikeIfssJMnZ5L%2FTLmCuX2DSyZ69LAZCpIpxG8gyez0O6%2F%2BBbC9lbb3TSKE4i5TcKjV%2FXrUIWPyoe4IT9zzd4%2BGYRiec2FX7J6mhYb046Kl2LWWEAZr0Awx%2Fr2dkCZcqya8VfVh1VgdAcidVLzXEzD9Ff8wf2y6p3d5Q%2FMnEMqfUkxc0Hw96hyLvDBOXWcm8xFy%2Bpn1Xi90ggqNmMceKqHs69bVYFFynprJiFtzum3IjOFjBpwQEGi7PJBrb22ux3eIfRAi50w86Djrx%2Fsj%2FpOTbpfNuDl2sg6oxmHbjzSEt9f6WGL0Tdld0Nhgd%2FZspKTqjKcdRwsJ0kpisj2XpzZTur2sRMyaf7Lrcgy1ZKm3CfYzDA4r7NBjqZAVxaI87IMhVLOtEzw9hVJ3NlFpdcV9yk1GGV9GUxJtAwxS%2F0m9PltdiaU6BNn%2BlsZQzuM9SBYOoB80kEQ5A6lBXgvQorXcmDQRjSXKWQxAgthWeqNAH%2BG6iTsblWm9avUTFRmzMH8gHrT9aLi%2FdSUh7js7gvO2Z7dVWZP%2F%2FQi3x%2BaJM1mNLx%2BSJ3KHL6kjgmeBljMffxiK1i0g%3D%3D&Expires=1773123097)

### Phase 7 – TLS & Certificate Lifecycle

- Issue origin certificate using Certbot DNS‑01 against SaaS Route53:
  - Validate SAN/CN = `<tenantSlug>-origin.openclaw-saas.io`.
  - Confirm full chain and trusted CA.
- Wire cert paths into Nginx `ssl_certificate` and `ssl_certificate_key`.
- Set up automated renewal:
  - Cron: `docker compose run --rm certbot renew` twice per day.
  - Deploy hook: `docker compose exec nginx nginx -s reload`.
- Add monitoring:
  - Script or CloudWatch check for days‑to‑expiry.
  - Alarm if expiry < 7 days or renewal error detected in `/var/log/certbot-renew.log`. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/21376642/a796a81a-f761-4d35-9b61-3d23e56c3ccf/Variant-A-Dockerized-Implementation-Only-CF.md.pdf?AWSAccessKeyId=ASIA2F3EMEYEW4EPXNFY&Signature=pAvf3Las94AdINcb69y0OoRenvA%3D&x-amz-security-token=IQoJb3JpZ2luX2VjEHYaCXVzLWVhc3QtMSJGMEQCIAiceeSAKm2Evm%2FTghzxLm7jtEMfqq9Irp2Vq%2B2nv7iVAiBrK22D6HLaCgYeqp%2B1g%2FQCp7y2ngekYn4gm6dhDj7KbSrzBAg%2FEAEaDDY5OTc1MzMwOTcwNSIMmsH3WZs8ZkaGyt%2BmKtAErzRg%2FqJcyd2IMxB3whwIHWwfHpCYj5xyFSx3P7ZYk7qExPqmuSqNL%2FIHEh4u0ZYObA9MexoY3JVwcdFC0ffFNoMvvSc917fcSnt5jZjU6fDLhiwBrj88OJ7357q4SW%2FH1dX1%2FXYJHKCrfQUjNUU230OQt9OWyh6aCQzzBNKoC%2BeXMj4FYEHqqPAgbwnQuC7MeGI%2Fk2Xrb4Fob2Uoo%2FZwzzVM00EuMIQyajQL5KKcmDHNxA4D4lBX4XwZNa4uJV%2B18r1wPOfMnqyQJ%2BseaCGktpXcYDjB3IhiEJCYCLYLit75FYK04LjJfrSwi8QBuCUtS0BimHtWsxMOJENsbK8rY6Li22EQu%2F92aUdqnr2CtylIZ3qolQUMHaVkyZ4zOzHZx4dIGwPyO7wrKfhxJw9aBOl28FNhfDxABmBvMYDTikeIfssJMnZ5L%2FTLmCuX2DSyZ69LAZCpIpxG8gyez0O6%2F%2BBbC9lbb3TSKE4i5TcKjV%2FXrUIWPyoe4IT9zzd4%2BGYRiec2FX7J6mhYb046Kl2LWWEAZr0Awx%2Fr2dkCZcqya8VfVh1VgdAcidVLzXEzD9Ff8wf2y6p3d5Q%2FMnEMqfUkxc0Hw96hyLvDBOXWcm8xFy%2Bpn1Xi90ggqNmMceKqHs69bVYFFynprJiFtzum3IjOFjBpwQEGi7PJBrb22ux3eIfRAi50w86Djrx%2Fsj%2FpOTbpfNuDl2sg6oxmHbjzSEt9f6WGL0Tdld0Nhgd%2FZspKTqjKcdRwsJ0kpisj2XpzZTur2sRMyaf7Lrcgy1ZKm3CfYzDA4r7NBjqZAVxaI87IMhVLOtEzw9hVJ3NlFpdcV9yk1GGV9GUxJtAwxS%2F0m9PltdiaU6BNn%2BlsZQzuM9SBYOoB80kEQ5A6lBXgvQorXcmDQRjSXKWQxAgthWeqNAH%2BG6iTsblWm9avUTFRmzMH8gHrT9aLi%2FdSUh7js7gvO2Z7dVWZP%2F%2FQi3x%2BaJM1mNLx%2BSJ3KHL6kjgmeBljMffxiK1i0g%3D%3D&Expires=1773123097)

### Phase 8 – CloudFront & WAF Setup

- Create CloudFront distribution:
  - Origin domain: `<tenantSlug>-origin.openclaw-saas.io`, HTTPS only, port 443.
  - Origin SSL protocol: TLSv1.2.
  - Viewer protocol: Redirect HTTP → HTTPS.
  - Methods: GET, HEAD, OPTIONS, PUT, POST, PATCH, DELETE.
  - Cache policy: “CachingDisabled”.
  - Origin request policy: forward Host, Upgrade, Connection, etc. for WebSockets.
  - Custom origin header: `X-Origin-Verify: <origin-secret>` (CloudFront overwrites any viewer value).
- Use default CloudFront domain like `d1234abcd.cloudfront.net` as viewer URL (later CNAME target).
- Create WAFv2 Web ACL (scope CLOUDFRONT, region us‑east‑1):
  - Add AWS managed rule sets (Common, KnownBadInputs, IP reputation, Anonymous IP).
  - Add rate‑based rule to limit per‑IP request rate.
  - Add custom rule to allow WebSocket Origin only from the CloudFront domain.
- Associate Web ACL with CloudFront distribution. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/21376642/a796a81a-f761-4d35-9b61-3d23e56c3ccf/Variant-A-Dockerized-Implementation-Only-CF.md.pdf?AWSAccessKeyId=ASIA2F3EMEYEW4EPXNFY&Signature=pAvf3Las94AdINcb69y0OoRenvA%3D&x-amz-security-token=IQoJb3JpZ2luX2VjEHYaCXVzLWVhc3QtMSJGMEQCIAiceeSAKm2Evm%2FTghzxLm7jtEMfqq9Irp2Vq%2B2nv7iVAiBrK22D6HLaCgYeqp%2B1g%2FQCp7y2ngekYn4gm6dhDj7KbSrzBAg%2FEAEaDDY5OTc1MzMwOTcwNSIMmsH3WZs8ZkaGyt%2BmKtAErzRg%2FqJcyd2IMxB3whwIHWwfHpCYj5xyFSx3P7ZYk7qExPqmuSqNL%2FIHEh4u0ZYObA9MexoY3JVwcdFC0ffFNoMvvSc917fcSnt5jZjU6fDLhiwBrj88OJ7357q4SW%2FH1dX1%2FXYJHKCrfQUjNUU230OQt9OWyh6aCQzzBNKoC%2BeXMj4FYEHqqPAgbwnQuC7MeGI%2Fk2Xrb4Fob2Uoo%2FZwzzVM00EuMIQyajQL5KKcmDHNxA4D4lBX4XwZNa4uJV%2B18r1wPOfMnqyQJ%2BseaCGktpXcYDjB3IhiEJCYCLYLit75FYK04LjJfrSwi8QBuCUtS0BimHtWsxMOJENsbK8rY6Li22EQu%2F92aUdqnr2CtylIZ3qolQUMHaVkyZ4zOzHZx4dIGwPyO7wrKfhxJw9aBOl28FNhfDxABmBvMYDTikeIfssJMnZ5L%2FTLmCuX2DSyZ69LAZCpIpxG8gyez0O6%2F%2BBbC9lbb3TSKE4i5TcKjV%2FXrUIWPyoe4IT9zzd4%2BGYRiec2FX7J6mhYb046Kl2LWWEAZr0Awx%2Fr2dkCZcqya8VfVh1VgdAcidVLzXEzD9Ff8wf2y6p3d5Q%2FMnEMqfUkxc0Hw96hyLvDBOXWcm8xFy%2Bpn1Xi90ggqNmMceKqHs69bVYFFynprJiFtzum3IjOFjBpwQEGi7PJBrb22ux3eIfRAi50w86Djrx%2Fsj%2FpOTbpfNuDl2sg6oxmHbjzSEt9f6WGL0Tdld0Nhgd%2FZspKTqjKcdRwsJ0kpisj2XpzZTur2sRMyaf7Lrcgy1ZKm3CfYzDA4r7NBjqZAVxaI87IMhVLOtEzw9hVJ3NlFpdcV9yk1GGV9GUxJtAwxS%2F0m9PltdiaU6BNn%2BlsZQzuM9SBYOoB80kEQ5A6lBXgvQorXcmDQRjSXKWQxAgthWeqNAH%2BG6iTsblWm9avUTFRmzMH8gHrT9aLi%2FdSUh7js7gvO2Z7dVWZP%2F%2FQi3x%2BaJM1mNLx%2BSJ3KHL6kjgmeBljMffxiK1i0g%3D%3D&Expires=1773123097)

### Phase 9 – CloudFront Pricing Mode

- If `pricingMode = "paygo"`:
  - Use default on‑demand pricing (no extra steps).
- If `pricingMode = "flat-rate-guided"`:
  - Implement preflight checks for eligibility.
  - Guide operator through AWS Console / API subscription for flat‑rate plan.
  - Re‑run validation after pricing change. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/21376642/a796a81a-f761-4d35-9b61-3d23e56c3ccf/Variant-A-Dockerized-Implementation-Only-CF.md.pdf?AWSAccessKeyId=ASIA2F3EMEYEW4EPXNFY&Signature=pAvf3Las94AdINcb69y0OoRenvA%3D&x-amz-security-token=IQoJb3JpZ2luX2VjEHYaCXVzLWVhc3QtMSJGMEQCIAiceeSAKm2Evm%2FTghzxLm7jtEMfqq9Irp2Vq%2B2nv7iVAiBrK22D6HLaCgYeqp%2B1g%2FQCp7y2ngekYn4gm6dhDj7KbSrzBAg%2FEAEaDDY5OTc1MzMwOTcwNSIMmsH3WZs8ZkaGyt%2BmKtAErzRg%2FqJcyd2IMxB3whwIHWwfHpCYj5xyFSx3P7ZYk7qExPqmuSqNL%2FIHEh4u0ZYObA9MexoY3JVwcdFC0ffFNoMvvSc917fcSnt5jZjU6fDLhiwBrj88OJ7357q4SW%2FH1dX1%2FXYJHKCrfQUjNUU230OQt9OWyh6aCQzzBNKoC%2BeXMj4FYEHqqPAgbwnQuC7MeGI%2Fk2Xrb4Fob2Uoo%2FZwzzVM00EuMIQyajQL5KKcmDHNxA4D4lBX4XwZNa4uJV%2B18r1wPOfMnqyQJ%2BseaCGktpXcYDjB3IhiEJCYCLYLit75FYK04LjJfrSwi8QBuCUtS0BimHtWsxMOJENsbK8rY6Li22EQu%2F92aUdqnr2CtylIZ3qolQUMHaVkyZ4zOzHZx4dIGwPyO7wrKfhxJw9aBOl28FNhfDxABmBvMYDTikeIfssJMnZ5L%2FTLmCuX2DSyZ69LAZCpIpxG8gyez0O6%2F%2BBbC9lbb3TSKE4i5TcKjV%2FXrUIWPyoe4IT9zzd4%2BGYRiec2FX7J6mhYb046Kl2LWWEAZr0Awx%2Fr2dkCZcqya8VfVh1VgdAcidVLzXEzD9Ff8wf2y6p3d5Q%2FMnEMqfUkxc0Hw96hyLvDBOXWcm8xFy%2Bpn1Xi90ggqNmMceKqHs69bVYFFynprJiFtzum3IjOFjBpwQEGi7PJBrb22ux3eIfRAi50w86Djrx%2Fsj%2FpOTbpfNuDl2sg6oxmHbjzSEt9f6WGL0Tdld0Nhgd%2FZspKTqjKcdRwsJ0kpisj2XpzZTur2sRMyaf7Lrcgy1ZKm3CfYzDA4r7NBjqZAVxaI87IMhVLOtEzw9hVJ3NlFpdcV9yk1GGV9GUxJtAwxS%2F0m9PltdiaU6BNn%2BlsZQzuM9SBYOoB80kEQ5A6lBXgvQorXcmDQRjSXKWQxAgthWeqNAH%2BG6iTsblWm9avUTFRmzMH8gHrT9aLi%2FdSUh7js7gvO2Z7dVWZP%2F%2FQi3x%2BaJM1mNLx%2BSJ3KHL6kjgmeBljMffxiK1i0g%3D%3D&Expires=1773123097)

### Phase 10 – Validation Gates (post‑deploy checks)

- Confirm CloudFront distribution status = Deployed.
- Confirm WAF Web ACL is attached and active.
- `docker compose ps openclaw-gateway` → Up (healthy).
- `docker compose ps nginx` → Up.
- `curl -fsS http://127.0.0.1:18789/healthz` from host or `docker compose exec` → HTTP 200.
- Verify origin certificate:
  - SAN matches origin domain, full chain, not expired.
- From the internet:
  - `https://d1234abcd.cloudfront.net` returns expected UI/API.
  - WebSocket upgrade to `wss://d1234abcd.cloudfront.net` succeeds.
- Ensure:
  - Direct origin access to EC2 IP is blocked.
  - Any request without correct `X-Origin-Verify` returns 403. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/21376642/a796a81a-f761-4d35-9b61-3d23e56c3ccf/Variant-A-Dockerized-Implementation-Only-CF.md.pdf?AWSAccessKeyId=ASIA2F3EMEYEW4EPXNFY&Signature=pAvf3Las94AdINcb69y0OoRenvA%3D&x-amz-security-token=IQoJb3JpZ2luX2VjEHYaCXVzLWVhc3QtMSJGMEQCIAiceeSAKm2Evm%2FTghzxLm7jtEMfqq9Irp2Vq%2B2nv7iVAiBrK22D6HLaCgYeqp%2B1g%2FQCp7y2ngekYn4gm6dhDj7KbSrzBAg%2FEAEaDDY5OTc1MzMwOTcwNSIMmsH3WZs8ZkaGyt%2BmKtAErzRg%2FqJcyd2IMxB3whwIHWwfHpCYj5xyFSx3P7ZYk7qExPqmuSqNL%2FIHEh4u0ZYObA9MexoY3JVwcdFC0ffFNoMvvSc917fcSnt5jZjU6fDLhiwBrj88OJ7357q4SW%2FH1dX1%2FXYJHKCrfQUjNUU230OQt9OWyh6aCQzzBNKoC%2BeXMj4FYEHqqPAgbwnQuC7MeGI%2Fk2Xrb4Fob2Uoo%2FZwzzVM00EuMIQyajQL5KKcmDHNxA4D4lBX4XwZNa4uJV%2B18r1wPOfMnqyQJ%2BseaCGktpXcYDjB3IhiEJCYCLYLit75FYK04LjJfrSwi8QBuCUtS0BimHtWsxMOJENsbK8rY6Li22EQu%2F92aUdqnr2CtylIZ3qolQUMHaVkyZ4zOzHZx4dIGwPyO7wrKfhxJw9aBOl28FNhfDxABmBvMYDTikeIfssJMnZ5L%2FTLmCuX2DSyZ69LAZCpIpxG8gyez0O6%2F%2BBbC9lbb3TSKE4i5TcKjV%2FXrUIWPyoe4IT9zzd4%2BGYRiec2FX7J6mhYb046Kl2LWWEAZr0Awx%2Fr2dkCZcqya8VfVh1VgdAcidVLzXEzD9Ff8wf2y6p3d5Q%2FMnEMqfUkxc0Hw96hyLvDBOXWcm8xFy%2Bpn1Xi90ggqNmMceKqHs69bVYFFynprJiFtzum3IjOFjBpwQEGi7PJBrb22ux3eIfRAi50w86Djrx%2Fsj%2FpOTbpfNuDl2sg6oxmHbjzSEt9f6WGL0Tdld0Nhgd%2FZspKTqjKcdRwsJ0kpisj2XpzZTur2sRMyaf7Lrcgy1ZKm3CfYzDA4r7NBjqZAVxaI87IMhVLOtEzw9hVJ3NlFpdcV9yk1GGV9GUxJtAwxS%2F0m9PltdiaU6BNn%2BlsZQzuM9SBYOoB80kEQ5A6lBXgvQorXcmDQRjSXKWQxAgthWeqNAH%2BG6iTsblWm9avUTFRmzMH8gHrT9aLi%2FdSUh7js7gvO2Z7dVWZP%2F%2FQi3x%2BaJM1mNLx%2BSJ3KHL6kjgmeBljMffxiK1i0g%3D%3D&Expires=1773123097)

### Phase 11 – Operations & Rotations

- Gateway token rotation:
  - Generate new token, update Secrets Manager, update `/opt/openclaw/.env`.
  - `cd /opt/openclaw && docker compose restart openclaw-gateway`.
  - Re‑run gateway health check.
- Origin secret rotation (zero‑downtime):
  - Implement dual‑accept in Nginx (temporarily accept old and new).
  - Update CloudFront origin custom header to new secret, wait for Deployed.
  - Remove old secret from Nginx map, reload Nginx.
  - Cleanup old secret version in Secrets Manager.
- Cert renewal failure runbook:
  - Inspect `docker compose logs certbot`.
  - Verify Route53 DNS permissions.
  - Force‑renew and reload Nginx.
- CloudFront 502 troubleshooting:
  - Check origin cert validity, Nginx status, SG rules, and CloudFront error headers.
- Monthly audits:
  - Review WAF logs, CloudTrail AssumeRole sessions, Secrets Manager access. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/21376642/a796a81a-f761-4d35-9b61-3d23e56c3ccf/Variant-A-Dockerized-Implementation-Only-CF.md.pdf?AWSAccessKeyId=ASIA2F3EMEYEW4EPXNFY&Signature=pAvf3Las94AdINcb69y0OoRenvA%3D&x-amz-security-token=IQoJb3JpZ2luX2VjEHYaCXVzLWVhc3QtMSJGMEQCIAiceeSAKm2Evm%2FTghzxLm7jtEMfqq9Irp2Vq%2B2nv7iVAiBrK22D6HLaCgYeqp%2B1g%2FQCp7y2ngekYn4gm6dhDj7KbSrzBAg%2FEAEaDDY5OTc1MzMwOTcwNSIMmsH3WZs8ZkaGyt%2BmKtAErzRg%2FqJcyd2IMxB3whwIHWwfHpCYj5xyFSx3P7ZYk7qExPqmuSqNL%2FIHEh4u0ZYObA9MexoY3JVwcdFC0ffFNoMvvSc917fcSnt5jZjU6fDLhiwBrj88OJ7357q4SW%2FH1dX1%2FXYJHKCrfQUjNUU230OQt9OWyh6aCQzzBNKoC%2BeXMj4FYEHqqPAgbwnQuC7MeGI%2Fk2Xrb4Fob2Uoo%2FZwzzVM00EuMIQyajQL5KKcmDHNxA4D4lBX4XwZNa4uJV%2B18r1wPOfMnqyQJ%2BseaCGktpXcYDjB3IhiEJCYCLYLit75FYK04LjJfrSwi8QBuCUtS0BimHtWsxMOJENsbK8rY6Li22EQu%2F92aUdqnr2CtylIZ3qolQUMHaVkyZ4zOzHZx4dIGwPyO7wrKfhxJw9aBOl28FNhfDxABmBvMYDTikeIfssJMnZ5L%2FTLmCuX2DSyZ69LAZCpIpxG8gyez0O6%2F%2BBbC9lbb3TSKE4i5TcKjV%2FXrUIWPyoe4IT9zzd4%2BGYRiec2FX7J6mhYb046Kl2LWWEAZr0Awx%2Fr2dkCZcqya8VfVh1VgdAcidVLzXEzD9Ff8wf2y6p3d5Q%2FMnEMqfUkxc0Hw96hyLvDBOXWcm8xFy%2Bpn1Xi90ggqNmMceKqHs69bVYFFynprJiFtzum3IjOFjBpwQEGi7PJBrb22ux3eIfRAi50w86Djrx%2Fsj%2FpOTbpfNuDl2sg6oxmHbjzSEt9f6WGL0Tdld0Nhgd%2FZspKTqjKcdRwsJ0kpisj2XpzZTur2sRMyaf7Lrcgy1ZKm3CfYzDA4r7NBjqZAVxaI87IMhVLOtEzw9hVJ3NlFpdcV9yk1GGV9GUxJtAwxS%2F0m9PltdiaU6BNn%2BlsZQzuM9SBYOoB80kEQ5A6lBXgvQorXcmDQRjSXKWQxAgthWeqNAH%2BG6iTsblWm9avUTFRmzMH8gHrT9aLi%2FdSUh7js7gvO2Z7dVWZP%2F%2FQi3x%2BaJM1mNLx%2BSJ3KHL6kjgmeBljMffxiK1i0g%3D%3D&Expires=1773123097)

### Phase 11b – SaaS‑Triggered Version Upgrades

- SaaS UI exposes “Upgrade OpenClaw to version X”.
- SaaS calls SSM `SendCommand` to EC2 to:
  - `cd /opt/openclaw`.
  - `docker compose pull openclaw-gateway`.
  - `docker compose up -d --no-deps openclaw-gateway`.
  - `docker image prune -f` to remove unused images.
  - Run health check on `http://127.0.0.1:18789/healthz`.
- Ensure no restart of Nginx during app upgrade (avoid TLS outage). [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/21376642/a796a81a-f761-4d35-9b61-3d23e56c3ccf/Variant-A-Dockerized-Implementation-Only-CF.md.pdf?AWSAccessKeyId=ASIA2F3EMEYEW4EPXNFY&Signature=pAvf3Las94AdINcb69y0OoRenvA%3D&x-amz-security-token=IQoJb3JpZ2luX2VjEHYaCXVzLWVhc3QtMSJGMEQCIAiceeSAKm2Evm%2FTghzxLm7jtEMfqq9Irp2Vq%2B2nv7iVAiBrK22D6HLaCgYeqp%2B1g%2FQCp7y2ngekYn4gm6dhDj7KbSrzBAg%2FEAEaDDY5OTc1MzMwOTcwNSIMmsH3WZs8ZkaGyt%2BmKtAErzRg%2FqJcyd2IMxB3whwIHWwfHpCYj5xyFSx3P7ZYk7qExPqmuSqNL%2FIHEh4u0ZYObA9MexoY3JVwcdFC0ffFNoMvvSc917fcSnt5jZjU6fDLhiwBrj88OJ7357q4SW%2FH1dX1%2FXYJHKCrfQUjNUU230OQt9OWyh6aCQzzBNKoC%2BeXMj4FYEHqqPAgbwnQuC7MeGI%2Fk2Xrb4Fob2Uoo%2FZwzzVM00EuMIQyajQL5KKcmDHNxA4D4lBX4XwZNa4uJV%2B18r1wPOfMnqyQJ%2BseaCGktpXcYDjB3IhiEJCYCLYLit75FYK04LjJfrSwi8QBuCUtS0BimHtWsxMOJENsbK8rY6Li22EQu%2F92aUdqnr2CtylIZ3qolQUMHaVkyZ4zOzHZx4dIGwPyO7wrKfhxJw9aBOl28FNhfDxABmBvMYDTikeIfssJMnZ5L%2FTLmCuX2DSyZ69LAZCpIpxG8gyez0O6%2F%2BBbC9lbb3TSKE4i5TcKjV%2FXrUIWPyoe4IT9zzd4%2BGYRiec2FX7J6mhYb046Kl2LWWEAZr0Awx%2Fr2dkCZcqya8VfVh1VgdAcidVLzXEzD9Ff8wf2y6p3d5Q%2FMnEMqfUkxc0Hw96hyLvDBOXWcm8xFy%2Bpn1Xi90ggqNmMceKqHs69bVYFFynprJiFtzum3IjOFjBpwQEGi7PJBrb22ux3eIfRAi50w86Djrx%2Fsj%2FpOTbpfNuDl2sg6oxmHbjzSEt9f6WGL0Tdld0Nhgd%2FZspKTqjKcdRwsJ0kpisj2XpzZTur2sRMyaf7Lrcgy1ZKm3CfYzDA4r7NBjqZAVxaI87IMhVLOtEzw9hVJ3NlFpdcV9yk1GGV9GUxJtAwxS%2F0m9PltdiaU6BNn%2BlsZQzuM9SBYOoB80kEQ5A6lBXgvQorXcmDQRjSXKWQxAgthWeqNAH%2BG6iTsblWm9avUTFRmzMH8gHrT9aLi%2FdSUh7js7gvO2Z7dVWZP%2F%2FQi3x%2BaJM1mNLx%2BSJ3KHL6kjgmeBljMffxiK1i0g%3D%3D&Expires=1773123097)

### Phase 12 – Deprovisioning

- Run `sst remove --stage <stage>` to destroy infrastructure in dependency order.
- Wait for CloudFront distribution deletion to complete (can take 15–30 minutes).
- Remove SaaS Route53 origin A record for `<tenantSlug>-origin.openclaw-saas.io`.
- Revoke/cancel Let’s Encrypt certificates if needed.
- Confirm no remaining billable resources (EC2, SGs, WAF, CloudFront, Secrets). [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/21376642/a796a81a-f761-4d35-9b61-3d23e56c3ccf/Variant-A-Dockerized-Implementation-Only-CF.md.pdf?AWSAccessKeyId=ASIA2F3EMEYEW4EPXNFY&Signature=pAvf3Las94AdINcb69y0OoRenvA%3D&x-amz-security-token=IQoJb3JpZ2luX2VjEHYaCXVzLWVhc3QtMSJGMEQCIAiceeSAKm2Evm%2FTghzxLm7jtEMfqq9Irp2Vq%2B2nv7iVAiBrK22D6HLaCgYeqp%2B1g%2FQCp7y2ngekYn4gm6dhDj7KbSrzBAg%2FEAEaDDY5OTc1MzMwOTcwNSIMmsH3WZs8ZkaGyt%2BmKtAErzRg%2FqJcyd2IMxB3whwIHWwfHpCYj5xyFSx3P7ZYk7qExPqmuSqNL%2FIHEh4u0ZYObA9MexoY3JVwcdFC0ffFNoMvvSc917fcSnt5jZjU6fDLhiwBrj88OJ7357q4SW%2FH1dX1%2FXYJHKCrfQUjNUU230OQt9OWyh6aCQzzBNKoC%2BeXMj4FYEHqqPAgbwnQuC7MeGI%2Fk2Xrb4Fob2Uoo%2FZwzzVM00EuMIQyajQL5KKcmDHNxA4D4lBX4XwZNa4uJV%2B18r1wPOfMnqyQJ%2BseaCGktpXcYDjB3IhiEJCYCLYLit75FYK04LjJfrSwi8QBuCUtS0BimHtWsxMOJENsbK8rY6Li22EQu%2F92aUdqnr2CtylIZ3qolQUMHaVkyZ4zOzHZx4dIGwPyO7wrKfhxJw9aBOl28FNhfDxABmBvMYDTikeIfssJMnZ5L%2FTLmCuX2DSyZ69LAZCpIpxG8gyez0O6%2F%2BBbC9lbb3TSKE4i5TcKjV%2FXrUIWPyoe4IT9zzd4%2BGYRiec2FX7J6mhYb046Kl2LWWEAZr0Awx%2Fr2dkCZcqya8VfVh1VgdAcidVLzXEzD9Ff8wf2y6p3d5Q%2FMnEMqfUkxc0Hw96hyLvDBOXWcm8xFy%2Bpn1Xi90ggqNmMceKqHs69bVYFFynprJiFtzum3IjOFjBpwQEGi7PJBrb22ux3eIfRAi50w86Djrx%2Fsj%2FpOTbpfNuDl2sg6oxmHbjzSEt9f6WGL0Tdld0Nhgd%2FZspKTqjKcdRwsJ0kpisj2XpzZTur2sRMyaf7Lrcgy1ZKm3CfYzDA4r7NBjqZAVxaI87IMhVLOtEzw9hVJ3NlFpdcV9yk1GGV9GUxJtAwxS%2F0m9PltdiaU6BNn%2BlsZQzuM9SBYOoB80kEQ5A6lBXgvQorXcmDQRjSXKWQxAgthWeqNAH%2BG6iTsblWm9avUTFRmzMH8gHrT9aLi%2FdSUh7js7gvO2Z7dVWZP%2F%2FQi3x%2BaJM1mNLx%2BSJ3KHL6kjgmeBljMffxiK1i0g%3D%3D&Expires=1773123097)

***

If you want, I can next turn this into a Mermaid diagram snippet you can paste into docs or an SST README.
