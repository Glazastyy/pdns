Using REGEX records
===================

REGEX records are PowerDNS-specific dynamic synthesis rules. They let an
authoritative zone answer names that match a regular expression without storing
each generated owner name as a separate record.

REGEX records are disabled by default. Enable them explicitly:

.. code-block:: ini

    regex-records=yes

The records must be stored at the zone apex. For a zone named
``example.com.``, create the REGEX record on ``example.com.`` itself, not on the
names that will be synthesized.

Syntax
------

The content of a REGEX record is:

::

    TYPE /pattern/ replacement

``TYPE``
  The DNS type to synthesize, for example ``A``, ``AAAA``, ``CNAME``, ``TXT``,
  ``MX`` or ``SRV``. ``REGEX``, ``RRSIG``, ``NSEC`` and ``NSEC3`` cannot be
  synthesized by REGEX rules.

``/pattern/``
  An ECMAScript regular expression matched case-insensitively against the
  absolute query name, including the trailing dot. Use anchors when you want an
  exact DNS name match.

``replacement``
  The record content to build when the pattern matches. Capture groups from the
  pattern can be used as ``$1``, ``$2`` and so on.

The delimiter does not have to be ``/``. The first character before the pattern
is used as the delimiter, so these are equivalent styles:

::

    A /^host-(\d+)\.example\.com\.$/ 192.0.2.$1
    A #^host-(\d+)\.example\.com\.$# 192.0.2.$1

If the delimiter appears inside the pattern, escape it with a backslash.

Examples
--------

Generate IPv4 addresses from numbered hostnames:

::

    $ORIGIN example.com.
    @ 3600 IN REGEX "A /^host-(\d+)\.example\.com\.$/ 192.0.2.$1"

``host-25.example.com. IN A`` returns ``192.0.2.25``.

Generate IPv6 addresses:

::

    @ 3600 IN REGEX "AAAA /^node-([0-9a-f]+)\.example\.com\.$/ 2001:db8::$1"

Return a CNAME and let PowerDNS continue resolution as it would for a normal
CNAME:

::

    @ 3600 IN REGEX "CNAME /^app-([a-z0-9-]+)\.example\.com\.$/ $1.apps.example.net."

Return TXT records:

::

    @ 3600 IN REGEX "TXT /^token-([a-z0-9]+)\.example\.com\.$/ verification=$1"

Because the replacement is parsed as normal content for the synthesized type,
use the regular zone-file representation for that type. For example, MX content
must include the preference and target:

::

    @ 3600 IN REGEX "MX /^mail-(\d+)\.example\.com\.$/ 10 mx$1.example.com."

HTTP API
--------

When creating REGEX records through the HTTP API, send the entire rule as one
record content string. For JSON, escape backslashes as usual:

.. code-block:: json

    {
      "rrsets": [
        {
          "name": "example.com.",
          "type": "REGEX",
          "ttl": 3600,
          "changetype": "REPLACE",
          "records": [
            {
              "content": "\"A /^host-(\\d+)\\.example\\.com\\.$/ 192.0.2.$1\"",
              "disabled": false
            }
          ]
        }
      ]
    }

Zone files and some APIs require the full rule to be quoted because it contains
spaces. PowerDNS accepts quoted rules and removes the outer quotes before
parsing the REGEX expression.

Query behavior
--------------

PowerDNS checks REGEX rules when no regular RRset directly answers the query, or
when the name exists but has no RRset of the requested type. Matching rules
synthesize answer records at the queried owner name.

Multiple matching REGEX records can synthesize multiple answers for the same
query. Rule order is backend-dependent, so avoid depending on ordering when more
than one rule can match the same name and type.

If a rule cannot be parsed, it is ignored. If the pattern or synthesized record
content is invalid while processing a matching rule, the query returns
``SERVFAIL`` and an error is logged.

DNSSEC
------

REGEX synthesis works with online-signed DNSSEC zones. Synthesized RRsets are
generated during query processing and are signed before the response is
returned.

REGEX synthesis is not performed for presigned zones, because PowerDNS cannot
precompute signatures for names that do not exist in the stored zone data.

The REGEX records themselves are hidden from normal answers in online-signed
zones. Queries receive the synthesized RRset instead of the rule.

Caching and performance
-----------------------

Synthesized REGEX answers are generated online and are marked as not
packet-cacheable. Keep patterns specific, anchored and cheap to evaluate. Prefer
ordinary records or wildcards for simple fixed cases, and use REGEX when capture
groups or structured name-to-content mapping are useful.

Testing
-------

After enabling ``regex-records`` and adding a rule, reload or rectify the zone
as you normally would for the backend in use. Then test with ``dig``:

.. code-block:: sh

    dig @127.0.0.1 host-25.example.com A
    dig @127.0.0.1 host-25.example.com A +dnssec

For DNSSEC zones, the ``+dnssec`` response should include signatures for the
synthesized answer when the zone is online-signed.

