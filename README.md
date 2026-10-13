# Countermeasures Against DNS Spoofing (Proof of Concept)

**Temat:** Demonstracja ataku DNS spoofing oraz metod obrony. Porównanie szyfrowania ruchu DNS (DoH) z integralnością ruchu (DNSSEC).
**Data:** 2026-10-13
**Środowisko:** w pełni izolowane laboratorium zbudowane w **EVE-NG** (node'y QEMU połączone wewnętrznym segmentem L2, bez atakowania cudzych systemów; dostęp do Internetu tylko przez kontrolowany node Cloud/NAT dla DoH).

---

## 1. Cel projektu (co chcemy zademonstrować)

1. **Atak:** pokazać, jak w sieci lokalnej atakujący może podmienić odpowiedź DNS (DNS spoofing / cache poisoning na poziomie LAN) i przekierować ofiarę na fałszywy adres.
2. **Obrona — poufność:** pokazać, że **DoH (DNS over HTTPS)** szyfruje zapytania DNS, przez co klasyczny podsłuch/podmiana w LAN przestaje działać (atakujący nie widzi i nie modyfikuje zapytań w locie).
3. **Obrona — integralność:** pokazać, że **DNSSEC** kryptograficznie podpisuje rekordy, więc sfałszowana odpowiedź zostaje **odrzucona** przez walidujący resolver (nawet jeśli ruch jest widoczny).
4. **Wniosek dydaktyczny:** DoH i DNSSEC rozwiązują **różne** problemy (poufność vs. integralność) i najlepiej działają razem.

---

## 2. Narzędzia (sprzęt i oprogramowanie)

### Sprzęt / platforma
- Host z **EVE-NG** (Community/Pro) — bare-metal lub w VM (zagnieżdżona wirtualizacja włączona).
- Zasoby: min. **8–16 GB RAM** (każdy node QEMU to osobna VM; wariant z Windows jest cięższy niż sam Linux).
- Brak dodatkowego sprzętu — cała topologia realizowana programowo w EVE-NG.

### Topologia w EVE-NG (node'y QEMU)
| Node | Rola | OS / obraz |
|------|------|------------|
| `attacker` | atakujący (ARP spoofing, MITM, `dns.spoof`) | Kali Linux |
| `victim` | ofiara (klient DNS/HTTP) | Ubuntu / Windows |
| `resolver` | serwer/forwarder DNS | Linux z dnsmasq |
| `SW` | segment L2 łączący węzły | wbudowany switch/bridge EVE-NG |
| `Cloud/NAT` | kontrolowana brama do Internetu (tylko dla DoH) | node Cloud (pnet) |

Wszystkie node'y w jednym segmencie L2 (wspólna domena rozgłoszeniowa → warunek działania ARP spoofingu). Wyjście do Internetu **tylko** przez node Cloud/NAT i tylko na czas Eksperymentu B (DoH).

> **Uwaga wdrożeniowa:** obrazy (Kali, Ubuntu, ew. Windows) trzeba wgrać do `/opt/unetlab/addons/qemu/` i uruchomić `/opt/unetlab/wrappers/unl_wrapper -a fixpermissions`. To typowo pierwsza rzecz do sprawdzenia, jeśli node nie startuje.

### Oprogramowanie
- **bettercap** (https://github.com/bettercap/bettercap) — ARP spoofing + moduł `dns.spoof` do podmiany odpowiedzi DNS.
- **dnsmasq 2.90** (https://thekelleys.org.uk/dnsmasq/doc.html) — lekki serwer/forwarder DNS pełniący rolę lokalnego resolvera ofiary; służy też do pokazania konfiguracji i logów.
- **dnscrypt-proxy** (https://github.com/DNSCrypt/dnscrypt-proxy) — lokalny proxy DNS realizujące **DoH/DNSCrypt** po stronie klienta (obrona — poufność).
- **Narzędzia pomocnicze:** `dig` / `drill` (weryfikacja odpowiedzi i flagi **AD** dla DNSSEC), `tcpdump` / Wireshark (podgląd, czy ruch jest jawny czy szyfrowany), prosty serwer WWW (strona „prawdziwa" vs „podstawiona").

---

## 3. Status Proof of Concept (co już sprawdzone)

- [ ] EVE-NG działa, obrazy QEMU wgrane (`fixpermissions` wykonane), node'y startują.
- [ ] Topologia (attacker/victim/resolver + SW + Cloud/NAT) złożona, łączność L2 potwierdzona (ping).
- [ ] dnsmasq skonfigurowany jako resolver, klient rozwiązuje nazwy.
- [ ] bettercap uruchamia się i wykonuje ARP spoofing w labie (test podstawowy modułu).
- [ ] dnscrypt-proxy instaluje się i łączy z resolverem DoH (test podstawowy).
- [ ] `dig` pokazuje flagę **AD** dla domeny z DNSSEC i jej brak dla niepodpisanej.

> Na spotkanie: przychodzę przekonany, że scenariusz da się wykonać — każde narzędzie zainstalowane i przetestowane w podstawowym zakresie.

---

## 4. Plan eksperymentów

**Eksperyment A — Atak (baseline, bez ochrony)**
1. Klient używa zwykłego DNS (plaintext, port 53) przez dnsmasq.
2. Atakujący: ARP spoofing + `dns.spoof` w bettercap → podmiana rekordu A wybranej domeny.
3. Efekt: klient trafia na serwer atakującego. W Wireshark widać jawne zapytania DNS.

**Eksperyment B — Obrona: DoH (poufność)**
1. Klient kieruje DNS przez dnscrypt-proxy (DoH).
2. Powtórzenie ataku z Eksperymentu A.
3. Efekt: ruch DNS jest szyfrowany (HTTPS) → atakujący nie widzi/nie podmienia zapytań. Pokazać różnicę w Wireshark (jawne vs. TLS).

**Eksperyment C — Obrona: DNSSEC (integralność)**
1. Walidujący resolver + domena podpisana DNSSEC.
2. Próba wstrzyknięcia sfałszowanej odpowiedzi.
3. Efekt: odpowiedź odrzucona (SERVFAIL / brak flagi AD) → ofiara nie zostaje przekierowana.

**Podsumowanie porównawcze**
- Tabela: atak skuteczny? / ruch widoczny? / fałszywka odrzucona? — dla wariantów: plaintext, DoH, DNSSEC, DoH+DNSSEC.

---

## 5. Pytania/ryzyka do omówienia na spotkaniu
- Czy demonstrować cache poisoning na poziomie resolvera, czy wystarczy podmiana w LAN (MITM)? (PoC skupia się na LAN — prostsze i powtarzalne.)
- Zakres: czy pokazać też DoT (DNS over TLS) obok DoH?
- Czy potrzebna własna strefa DNSSEC w labie, czy użyć gotowej podpisanej domeny testowej.
