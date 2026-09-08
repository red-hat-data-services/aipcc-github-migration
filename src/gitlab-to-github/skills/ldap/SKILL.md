---
name: ldap
description: Use when you need to look up Red Hat people and groups in LDAP via ldapsearch — find someone's email, uid, full name, or GitHub/GitLab username; list a group's members or owners; find which groups a user belongs to; or translate a GitLab username to a GitHub username (e.g. for CODEOWNERS). Information in "Rover Groups" is also in LDAP.
allowed-tools: Bash, Read
---

# Red Hat LDAP lookups

Use `ldapsearch` against Red Hat's IPA LDAP to look up users and groups.

## Connection

Host: `ldap:///dc=ipa%2Cdc=redhat%2Cdc=com`

Authentication: GSSAPI (`-Q` flag). The user must have a valid Kerberos ticket (`kinit`).

Always use `-ZZ` (require StartTLS) on every query — never run `ldapsearch` without it.

Always use `-LLL` for clean LDIF output (no comments, no version line).

## Base DNs

- Users: `cn=users,cn=accounts,dc=ipa,dc=redhat,dc=com`
- Groups: `cn=groups,cn=accounts,dc=ipa,dc=redhat,dc=com`

## User lookups

Base DN: users.

| Goal | Filter | Attributes to request |
|---|---|---|
| Find user by uid | `(uid=kdreyer)` | `mail`, `cn`, `rhatSocialURL`, or omit for all |
| Find user by email | `(mail=kdreyer@redhat.com)` | any |
| Find user by GitLab username | `(rhatSocialURL=Gitlab->https://gitlab.com/USERNAME)` | `mail`, `uid`, `cn` |
| Find user by GitHub username | `(rhatSocialURL=Github->https://github.com/USERNAME)` | `mail`, `uid`, `cn` |
| Get a user's GitHub/GitLab usernames | `(uid=USERNAME)` | `rhatSocialURL` |

Example — find email from GitLab username:
```
ldapsearch -ZZ -H "ldap:///dc=ipa%2Cdc=redhat%2Cdc=com" -QLLL \
  -b 'cn=users,cn=accounts,dc=ipa,dc=redhat,dc=com' \
  '(rhatSocialURL=Gitlab->https://gitlab.com/kdreyer)' mail
```

## Group lookups

Base DN: groups.

### List members of a group

Filter: `(cn=GROUP_NAME)` — request `member` attribute. To also see group owners, request `owner`.

Example — list members and owners of a group:
```
ldapsearch -ZZ -H "ldap:///dc=ipa%2Cdc=redhat%2Cdc=com" -QLLL \
  -b "cn=groups,cn=accounts,dc=ipa,dc=redhat,dc=com" \
  "(cn=aaet-llama-stack-core-team)" member owner
```

The `member` and `owner` attributes return full DNs like `uid=kdreyer,cn=users,...`. To extract just uids, pipe through:
```
grep -E '^(member|owner):' | sed 's/.*: uid=\([^,]*\),.*/\1/'
```

### Find which groups a user belongs to

Search the groups base DN with a `member` filter matching the user's full DN:
```
ldapsearch -ZZ -H "ldap:///dc=ipa%2Cdc=redhat%2Cdc=com" -QLLL \
  -b "cn=groups,cn=accounts,dc=ipa,dc=redhat,dc=com" \
  "(member=uid=USERNAME,cn=users,cn=accounts,dc=ipa,dc=redhat,dc=com)" cn
```

To find groups where a user is an owner (not just a member):
```
ldapsearch -ZZ -H "ldap:///dc=ipa%2Cdc=redhat%2Cdc=com" -QLLL \
  -b "cn=groups,cn=accounts,dc=ipa,dc=redhat,dc=com" \
  "(owner=uid=USERNAME,cn=users,cn=accounts,dc=ipa,dc=redhat,dc=com)" cn
```

## Useful attributes

- `uid` — Red Hat Kerberos/login username
- `mail` — email address
- `cn` — full name
- `rhatSocialURL` — social links (format: `Github->https://github.com/USER`, `Gitlab->https://gitlab.com/USER`)
- `member` — (on groups) list of member DNs
- `owner` — (on groups) list of owner DNs — a user can be both owner and member

## "devel" LDAP group

The "devel" LDAP group is a an older POSIX-style LDAP group that historically has most of Red Hat engineering. It was used for ancient systems like dist-git, etc. Today it maps to the [identically-named "devel" GitLab group](https://gitlab.cee.redhat.com/groups/devel), which incidentally has ["Developer" permissions on konflux-release-data](https://gitlab.cee.redhat.com/releng/konflux-release-data/-/project_members?tab=groups). To join it, see [KB0003131](https://hub.redhat.com/hub?id=kb_article&sysparm_article=KB0003131).

## Notes

- If a query returns nothing, the user may need to run `kinit` first.
