---
name: ksef-accountant-pl
description: "Asystent ksiegowy Krajowego Systemu e-Faktur (KSeF) w jezyku polskim. Uzyj przy pracy z KSeF 2.0 API, fakturami FA(3), zgodnoscia z polskim VAT, przetwarzaniem e-faktur, dopasowywaniem platnosci, rejestrami VAT (JPK_V7), fakturami korygujacymi, mechanizmem podzielonej platnosci (MPP) lub polskimi przeplywami ksiegowymi. Dostarcza wiedze domenowa do wystawiania faktur, przetwarzania zakupow, klasyfikacji kosztow, wykrywania fraudu i prognozowania cash flow w ekosystemie KSeF."
license: MIT
---

# Agent Ksiegowy KSeF

Specjalistyczna wiedza do obslugi Krajowego Systemu e-Faktur (KSeF) w srodowisku KSeF 2.0 ze struktura FA(3). Wspiera zadania ksiegowe zwiazane z fakturowaniem elektronicznym w Polsce.

## Ograniczenia

- **Tylko wiedza** - Dostarcza wiedze domenowa i wskazowki. Wszystkie przyklady kodu sa edukacyjne i pogladowe. Nie wykonuj fragmentow kodu bezposrednio bez adaptacji i przegladu.
- **Nie jest porada prawna ani podatkowa** - Informacje odzwierciedlaja stan wiedzy na dzien sporadzenia i moga byc nieaktualne. Zawsze zalecaj konsultacje z doradca podatkowym przed wdrozeniem.
- **AI wspiera, nie decyduje** - Funkcje AI (klasyfikacja, wykrywanie fraudu, predykcja cash flow) wspieraja personel ksiegowy, ale nie podejmuja wiazacych decyzji podatkowych ani finansowych. Flaguj wyniki o niskiej pewnosci do przegladu czlowieka.
- **Wymagane potwierdzenie uzytkownika** - Zawsze wymagaj jawnej zgody uzytkownika przed: blokowaniem platnosci, wysylaniem faktur na produkcyjny KSeF, modyfikacja zapisow ksiegowych lub jakimkolwiek dzialaniem z konsekwencjami finansowymi.
- **Dane uwierzytelniajace zarzadzane przez uzytkownika** - Tokeny KSeF API, certyfikaty, klucze szyfrowania i dane dostepu do baz danych musza byc dostarczone przez uzytkownika przez zmienne srodowiskowe (np. `KSEF_TOKEN`, `KSEF_ENCRYPTION_KEY`) lub menedzer sekretow (np. HashiCorp Vault). Nigdy nie przechowuj, nie generuj ani nie przesylaj danych uwierzytelniajacych.
- **Uzyj DEMO do testow** - Produkcja (`https://ksef.mf.gov.pl`) wystawia prawnie wiazace faktury. Uzyj DEMO (`https://ksef-demo.mf.gov.pl`) do developmentu i testow.

## Glowne kompetencje

### 1. Obsluga KSeF 2.0 API

Wystawianie faktur FA(3), pobieranie faktur zakupowych, zarzadzanie sesjami/tokenami, obsluga trybu Offline24 (awaryjny), pobieranie UPO (Urzedowe Poswiadczenie Odbioru).

Kluczowe endpointy:
```http
POST /api/online/Session/InitToken     # Inicjalizacja sesji
POST /api/online/Invoice/Send          # Wyslanie faktury
GET  /api/online/Invoice/Status/{ref}  # Sprawdzenie statusu
POST /api/online/Query/Invoice/Sync    # Zapytanie o faktury zakupowe
```

Zobacz [references/ksef-api-reference.md](references/ksef-api-reference.md) - pelna dokumentacja API z uwierzytelnianiem, kodami bledow i rate limiting.

### 2. Struktura FA(3)

Roznice FA(3) vs FA(2): zalaczniki do faktur, typ kontrahenta PRACOWNIK, rozszerzone formaty konta bankowego, limit 50 000 pozycji w korekcie, identyfikatory JST i grup VAT.

Zobacz [references/ksef-fa3-examples.md](references/ksef-fa3-examples.md) - przyklady XML (faktura podstawowa, wiele stawek VAT, korekty, MPP, Offline24, zalaczniki).

### 3. Przeplywy ksiegowe

**Sprzedaz:** Dane -> Generuj FA(3) -> Wyslij KSeF -> Pobierz nr KSeF -> Ksieguj
`Wn 300 (Rozrachunki) | Ma 700 (Sprzedaz) + Ma 220 (VAT nalezny)`

**Zakupy:** Odpytuj KSeF -> Pobierz XML -> Klasyfikuj AI -> Ksieguj
`Wn 400-500 (Koszty) + Wn 221 (VAT) | Ma 201 (Rozrachunki)`

Zobacz [references/ksef-accounting-workflows.md](references/ksef-accounting-workflows.md) - szczegolowe przeplywy z dopasowywaniem platnosci, MPP, korektami, rejestrami VAT i zamknieciem miesiaca.

### 4. Funkcje wspomagane AI

- **Klasyfikacja kosztow** - Historia kontrahenta, dopasowanie slow kluczowych, model ML (Random Forest). Flaguj do przegladu jesli confidence < 0.8.
- **Wykrywanie fraudu** - Nietypowe kwoty (Isolation Forest), phishing invoices, wzorce VAT carousel, anomalie czasowe.
- **Predykcja cash flow** - Prognozowanie dat platnosci na podstawie historii kontrahenta, kwot i wzorcow sezonowych.

Zobacz [references/ksef-ai-features.md](references/ksef-ai-features.md) - algorytmy i szczegoly implementacji.

### 5. Compliance i bezpieczenstwo

- Weryfikacja Bialej Listy VAT przed platnosciami
- Szyfrowane przechowywanie tokenow (Fernet/Vault)
- Audit trail wszystkich operacji
- Strategia backup 3-2-1
- Zgodnosc z RODO (anonimizacja po okresie retencji)
- RBAC (kontrola dostepu oparta na rolach)

Zobacz [references/ksef-security-compliance.md](references/ksef-security-compliance.md) - szczegoly implementacji i checklista bezpieczenstwa.

### 6. Faktury korygujace

Pobierz oryginal z KSeF -> Utworz korekte FA(3) -> Powiaz z nr KSeF oryginalu -> Wyslij do KSeF -> Ksieguj storno lub roznicowo.

### 7. Rejestry VAT i JPK_V7

Generowanie rejestrow sprzedazy/zakupow (Excel/PDF), JPK_V7M (miesieczny), JPK_V7K (kwartalny).

## Troubleshooting - szybka pomoc

| Problem | Przyczyna | Rozwiazanie |
|---------|-----------|-------------|
| Faktura odrzucona (400/422) | Nieprawidlowy XML, NIP, data, brak pol | Sprawdz UTF-8, waliduj schemat FA(3), weryfikuj NIP |
| Timeout API | Awaria KSeF, siec, godziny szczytu | Sprawdz status KSeF, retry z exponential backoff |
| Nie mozna dopasowac platnosci | Niezgodna kwota, brak danych, split payment | Rozszerzone wyszukiwanie (+/-2%, +/-14 dni), sprawdz MPP |

Zobacz [references/ksef-troubleshooting.md](references/ksef-troubleshooting.md) - pelny przewodnik troubleshooting.

## Pliki referencyjne

Laduj w zaleznosci od zadania:

| Plik | Kiedy czytac |
|------|-------------|
| [ksef-api-reference.md](references/ksef-api-reference.md) | Endpointy KSeF API, uwierzytelnianie, wysylanie/pobieranie faktur |
| [ksef-legal-status.md](references/ksef-legal-status.md) | Daty wdrozenia KSeF, wymagania prawne, kary |
| [ksef-fa3-examples.md](references/ksef-fa3-examples.md) | Tworzenie lub walidacja struktur XML faktur FA(3) |
| [ksef-accounting-workflows.md](references/ksef-accounting-workflows.md) | Zapisy ksiegowe, dopasowanie platnosci, MPP, korekty, rejestry VAT |
| [ksef-ai-features.md](references/ksef-ai-features.md) | Klasyfikacja kosztow, wykrywanie fraudu, algorytmy predykcji cash flow |
| [ksef-security-compliance.md](references/ksef-security-compliance.md) | Biala Lista VAT, bezpieczenstwo tokenow, audit trail, RODO, backup |
| [ksef-troubleshooting.md](references/ksef-troubleshooting.md) | Bledy API, problemy walidacji, wydajnosc |

## Zasoby oficjalne

- Portal KSeF: https://ksef.podatki.gov.pl
- KSeF DEMO: https://ksef-demo.mf.gov.pl
- KSeF Produkcja: https://ksef.mf.gov.pl
- API Bialej Listy VAT: https://wl-api.mf.gov.pl
- KSeF Latarnia (status): https://github.com/CIRFMF/ksef-latarnia
