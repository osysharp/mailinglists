# Osysharp.MailingLists

Mailing lists any app can keep — a newsletter, an offers list, a changelog — done the way the law and the inbox
providers expect:

- **Double opt-in.** Signing up makes an address *waiting*; the owner of the mailbox confirms it from a mail, and
  nothing goes to it before they do. Typing someone else's address costs them one mail they can ignore — and at most
  one an hour.
- **Consent is a record, word for word.** Every join, confirmation and departure is a row: when, where (`"footer"`,
  `"checkout"`, `"account"`, `"unsubscribe link"`) and the exact words beside the box they ticked. That is what a
  data-protection authority asks for, and what nobody can reconstruct afterwards.
- **Links, no sign-in.** Every mail carries its reader's own link — to a page that shows which lists they are on, with
  a switch for each — and the one-click `List-Unsubscribe` / `List-Unsubscribe-Post` headers Gmail and Yahoo require on
  bulk mail. The links are platform **grants** (`ListReaderLink`, `ListConfirmation`): signed with the app's own key,
  one reader each, one purpose each (a confirm link cannot unsubscribe, a forged or edited one does nothing), checked by
  the platform before the page opens or the kit runs, and taken back when the reader is erased. What they ask for is
  done by the reader's own workflow, so a visitor holding one changes that one reader and nothing else.
- **Bounces and complaints stop an address.** Tell the kit what the mail provider's webhook said (`StopAddress`) and
  nothing is sent there again until its owner confirms it afresh.
- **Sending through your own outbox.** A campaign goes out one mail per confirmed reader through the app's outbox —
  retried there — each with its own way out, and a send run twice mails nobody twice.
- **Export and erasure.** Staff download a list as CSV (with each reader's consent), and erase a reader: address,
  name, lists, consents and what was sent to them.

## Use it

```osy
// app.osy
use Osysharp.MailingLists@0;
```

```osy
// model/lists.osy
app.MailingLists = new MailingListSetup {
  Lists = [ new ListInfo { Key = "letter", Name = "The monthly letter", Description = "New things, once a month." } ],
  Brand = "Acme", Address = "1 Main St, Springfield",
  Mailer = new AcmeListMailer(),               // hands each mail to your outbox — see below
};

// Who runs the lists, and which address is a signed-in reader's own, are yours to say — the kit cannot name your roles
// or your principal. One line each; unanswered, nobody runs the lists and no account reads an address.
policy RunsMailingLists => IsStaff;
policy OwnsListAddress(string email) => email == user.Email;
```

```osy
// the outbox: one row per mail, delivered (and retried) by your own workflow
public class AcmeListMailer : ListMailer {
  public override void Send(ListMailOut mail) {
    // mail.To, mail.Mail (subject, HTML, text), mail.Unsubscribe (the one-click address, "" for a confirmation),
    // mail.Key (the same mail handed over twice has the same key)
  }
}
```

With Resend, send the one-click pair with the mail: `Headers = ListUnsubscribeHeaders(mail.Unsubscribe)`.

```osy
// the pages the mails link to — the paths are MailingListSetup's, change them there. The parameters are the kit's
// grants, so a link that does not check out never opens the page: the visitor gets InvalidLinkSurface instead.
[Route("/newsletter/confirm/{link}")] [AllowAnonymous] component Confirm(ListConfirmation link) { render { ListConfirm(link); } }
[Route("/newsletter/{link}")] [AllowAnonymous] component Preferences(ListReaderLink link) { render { ListPreferences(link); } }
[Route("/newsletter/link-gone")] [AllowAnonymous] component LinkGone() { render { Text("That link has run out."); } }
app.Ui = new AppUi { InvalidLinkSurface = LinkGone };

// the one-click unsubscribe a mail client POSTs (RFC 8058) — an API route of yours:
new RestApi("Lists") { Route = "lists", Auth = new ApiAuth { Anonymous = true },
  Endpoints = [ new Endpoint(OneClickUnsubscribe) { Method = HttpMethod.Post, Path = "/leave/{list}/{link}", Raw = true } ] }
```

Components: `ListSignupForm(listKey, words, source:)` (a form; the words beside the button are what is recorded),
`ListConfirm(link)`, `ListPreferences(link)` (the reader's address and a switch per list), `ListChoices(email)` (a
signed-in reader's switches) and
`DeskMailingLists()` (staff: counts, readers, CSV, erase).

From code: `SignUpToList(email, listKey, source, words)`, `ConfirmSubscription(ListConfirmation link)`,
`ReaderChoices(ListReaderLink link)`, `ReaderAddress(link)`, `LeaveWithLink(link, listKey)`, `JoinWithLink(link,
listKey, words)`, `OneClickUnsubscribe(link, listKey)`, `PreferencesLink(reader)` / `OneClickLink(reader, listKey)`
(the URLs a mail carries), `ListChoicesFor(email)`, `ChooseList(email,
listKey, on, source, words)`; for staff `ReadersOf(listKey)`, `ListSummaries()`, `ListReaders(listKey)`,
`ExportReaders(listKey)`, `ReaderRecord(email)`, `EraseReader(email)`, `StopReader(reader, why)`; for the webhook
`StopAddress(email, complained, detail)`; and to send, derive from `ListCampaign` and call `SendToList(campaign,
listKey)`.

## How it is built

`ListSubscriber` is one row per address, whose `State` belongs to `SubscriberLifecycle` (waiting → confirmed ↔
stopped, erased). Every change from outside the app — a confirm or unsubscribe link, the provider's word — is
**deposited into that workflow by id or address** (`[CorrelateOn]`), whose body writes as the app: so no write can be
aimed at a row. A reader's link READS exactly one thing — its reader and that reader's memberships (`allow read by
ListReaderLink`), which is what lets the preferences page show where they stand. A sign-up arrives through `ListSignup`, a door any visitor may
open and nobody reads back, handled by its own workflow — which is why "already subscribed" is never said to a
stranger. `ListDelivery` (one row per campaign and reader, unique) is what makes a send safe to repeat.

The kit grants no reader of its own beyond the signed links: staff and signed-in readers come in through the app's two
answers (`RunsMailingLists`, `OwnsListAddress`). Every table is `[Authority(Extensible)]`, so an app may add finer
`partial entity` grants of its own on top.
