Factors
- TOTP (RFC6238)
- Out-of-band (SMS)
- FIDO U2F (Considered legacy)
- FIDO2 and WebAuthn

~~Always require MFA~~

Skip MFA for the user agent via cookie
And able to force MFA for the next login
=> bearer

Recovery codes
Consumable
Re-generate when one is consumed

ID Token
Set "amr" claim

Authenticator
- recoverycode
- totp
- oob
- bearer


/mfa/challenge
> Trigger a challenge. For example, trigger an OOB challenge like sending SMS

/login
{
  "error": "mfa_required",
  "mfa_token": "..."
}

# Associate TOTP
POST /mfa/associate
Authorization: Bearer <mfa_token>

{
  "authenticator_type": "totp"
}

{
  "authenticator_type": "totp",
  "secret": "...",
  "recovery_codes": ["...", "..."]
}

# Associate OOB
POST /mfa/associate
Authorization: Bearer <mfa_token>

{
  "authenticator_type": "oob",
  "oob_channel": "sms",
  "phone": "+852........"
}

{
  "authenticator_type": "oob",
  "oob_channel": "sms",
  "recovery_codes": ["...", "..."]
}

# Authenticate with solved challenge
POST /mfa/authenticate
Authorization: Bearer <mfa_token>

{
  "authenticator_type": "totp",
  "otp": "..."
}

# List authenticators
GET /mfa/authenticator
