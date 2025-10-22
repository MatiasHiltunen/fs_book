
# FullStack 1 - Mini Chat -projekti (React Router 7 + SSR)

Tässä tehtävässä rakennetaan reaaliaikainen chat-sovellus Supabasen avulla ja käytetään React Routerin framework-tilaa (versio 7) sekä palvelinpuolen renderöintiä (SSR). Aloituspohja on luotu `npx create-react-router@latest` -komennolla ja käyttää TypeScriptiä. Tehtävät ovat pisteytetty yhteensä **70 pisteen** arvosta, lisäksi extrana **10 bonus­pistettä** [React Routeriin](https://reactrouter.com/start/modes#framework) liittyen.

_Tehtävässä rakennettavaa mini-projektia tullaan hyödyntämään tehtäväsarjassa 3 tekoälyratkaisun kanssa!_



## 1. Projektin luonti ja alustaminen (5 p)

1. **Luo uusi React Router ‑projekti.**


Suorita:

```sh
npx create-react-router@latest my-chat-app --typescript
```
_Vaihda my-chat-app vastaamaan haluamaasi nimeä chatille_

Seuraa CLI:n ohjeita. Valitse "framework"-tila ja varmista, että `react-router.config.ts`‑tiedostossa on `ssr: true`. Tämä mahdollistaa palvelinpuolen renderöinnin.

2. **Asenna riippuvuudet ja käynnistä sovellus.**

   ```sh
   cd my-chat-app
   npm install
   npm run dev
   ```


3. **Konfiguroi Supabase.** Kopioi `.env.example` tiedosto projektin juureen nimellä `.env`. Täytä sinne oman Supabase-projektisi URL (`VITE_SUPABASE_URL`) ja anonyymi avain (`VITE_SUPABASE_ANON_KEY`). 

## 2. Kansiorakenne (5 p)

* `app/root.tsx` Toimii entrypointtina / layoutina App.jsx:n sijasta, sis. virheenkäsittelyn. Sisältää mm. `<Meta/>`, `<Links/>`, `<ScrollRestoration/>` ja `<Scripts/>`‑komponentit. Juuri­komponentti renderöi `<Outlet/>` komponentin sisällön, johon reitteihin kytketyt komponentit renderöidään.
* `app/routes.ts` määrittelee reitit. Jokaisella reitillä on URL-polku ja moduli­tiedosto. Esimerkissä on vain indeksi (`home.tsx`).
* `app/routes/home.tsx` on sovelluksen etusivu, joka renderöi `Welcome`‑komponentin.
* `app/welcome/welcome.tsx` sisältää tervetuloa‑näkymän. Tämä näkymä linkittää ulkoisiin resursseihin mutta ei vielä chat-sovellukseen.
* `react-router.config.ts` React-routerin konfiguraatiotiedostossa voidaan kytkeä SSR-tila päälle tai pois `ssr: true`.

## 3. Lisää Supabase-client (5 p)

1. Luo kansio `app/services` ja lisää sinne tiedosto `supabase.ts`.
2. Viedään tiedostosta `createClient`-funktion avulla luotu supabase-client instanssi

```ts
// app/services/supabase.ts
import { createClient } from '@supabase/supabase-js';

// Luo Supabase-asiakas. Lisätietoja: https://supabase.com/docs/reference/javascript/initialize
const supabaseUrl = import.meta.env.VITE_SUPABASE_URL;
const supabaseKey = import.meta.env.VITE_SUPABASE_ANON_KEY;

const supabase = createClient(supabaseUrl, supabaseKey);

export default supabase;
```

## 4. Luo chat-reitti tiedostopohjaisesti (10 p)

1. **Päivitä `app/routes.ts`** lisäämällä chat-polku. Käytä `route("chat", "./routes/chat.tsx")`:

   ```ts
   import { type RouteConfig, index, route } from "@react-router/dev/routes";

   export default [
     index("routes/home.tsx"),
     route("chat", "routes/chat.tsx"),
   ] satisfies RouteConfig;
   ```

2. **Luo `app/routes/chat.tsx`**, joka toimii sekä datalähteenä että UI-komponenttina:

   ```tsx
   import { useEffect, useRef, useState } from 'react';
   import { useLoaderData } from 'react-router';
   import type { Route } from './+types/chat';
   import supabase from '../services/supabase';

   // Lataa viestit palvelinpuolella. Lisätietoja loader-funktiosta: https://reactrouter.com/start/framework/data-loading
   export async function loader({}: Route.LoaderArgs) {
     const { data, error } = await supabase
       .from('messages')
       .select('*')
       .order('created_at', { ascending: true });
     if (error) {
       throw new Error(error.message);
     }
     return { messages: data };
   }

   // Pääkomponentti renderöi chat-näkymän. loaderData sisältää viestit.
   export default function Component({ loaderData }: Route.ComponentProps) {
     const [messages, setMessages] = useState(loaderData.messages);
     const [content, setContent] = useState('');
     const scrollRef = useRef<HTMLDivElement>(null);
     const username = 'Testaaja'; // TODO: anna käyttäjän syöttää nimensä esim. promptilla

     // Lisää uusi viesti Supabaseen ja paikalliseen tilaan
     async function addMessage() {
       if (!content.trim()) return;
       const { data, error } = await supabase
         .from('messages')
         .insert([{ username, content }])
         .select();
       if (error || !data) {
         alert('Virhe viestin lähetyksessä');
         return;
       }
       setMessages((curr) => [...curr, data[0]]);
       setContent('');
     }

     // Kuuntele postgres_changes-viestejä reaaliaikaisesti.
     useEffect(() => {
       const channel = supabase
         .channel('chat-room')
         .on(
           'postgres_changes',
           { event: 'INSERT', schema: 'public', table: 'messages' },
           (payload) => {
             // Älä lisää viestiä kahdesti, jos se on käyttäjän oma
             if (payload.new.username !== username) {
               setMessages((curr) => [...curr, payload.new]);
             }
           },
         )
         .subscribe();
       return () => {
         supabase.removeChannel(channel);
       };
     }, []);

     // Scrollaa alareunaan aina kun viestilista päivittyy
     useEffect(() => {
       if (scrollRef.current) {
         scrollRef.current.scroll({
           top: scrollRef.current.scrollHeight,
           behavior: 'smooth',
         });
       }
     }, [messages]);

     return (
       <main className="flex flex-col items-center gap-4 p-4">
         <h1 className="text-2xl font-bold">Chat</h1>
         <div
           ref={scrollRef}
           className="h-[40dvh] w-full max-w-[600px] overflow-y-auto border rounded p-2 space-y-2"
         >
           {messages.map((message: any) => {
             const isOwn = message.username === username;
             const timestamp = new Date(
               message.created_at ?? message.ts,
             ).toLocaleString('fi');
             return (
               <div
                 key={message.id ?? message.ts}
                 className={`chat ${isOwn ? 'chat-end' : 'chat-start'}`}
               >
                 <div className="chat-header">
                   {message.username}
                   <time className="text-xs opacity-50 ml-2">{timestamp}</time>
                 </div>
                 <div className="chat-bubble">{message.content}</div>
               </div>
             );
           })}
         </div>
         <textarea
           className="textarea w-full max-w-[600px]"
           placeholder="Kirjoita viesti"
           value={content}
           onChange={(e) => setContent(e.target.value)}
         ></textarea>
         <div className="flex w-full max-w-[600px] justify-end">
           <button
             onClick={addMessage}
             className="btn btn-primary"
             disabled={!content.trim()}
           >
             Send
           </button>
         </div>
       </main>
     );
   }
    ```

Chat-moduuli esittelee `loader`-funktion, joka suoritetaan palvelimella ja pääkomponentin, joka renderöi UI:n ja hoitaa reaaliaikaiset tilaukset Supabaseen. Reaaliaikainen kuuntelu käyttää `postgres_changes`-tapahtumaa.

## 5. Lisää navigointi chat-sivulle (5 p)

* Lisää `Welcome`-komponenttiin linkki `/chat`, jotta käyttäjät pääsevät chattiin. Voit käyttää `<Link to="/chat">Chat</Link>`-elementtiä React Routerin `react-router`-kirjastosta.
* Vaihtoehtoisesti lisää linkki suoraan `app/root.tsx`-layoutiin, jos haluat pysyvän navigaatiopalkin.

## 6. Viestien lähetys ja reaaliaikaisuus (15 p)

* **Hae viestit loaderissa.** Olet jo toteuttanut `loader`-funktion, joka hakee viestit Supabasesta.
* **Lähetä viesti.** `addMessage` kutsuu `supabase.from('messages').insert(...)` ja päivittää tilan.
* **Kuuntele reaaliaikaiset lisäykset.** `useEffect`-koukku tilaa `postgres_changes`-tapahtumaan Supabasen Realtime-kanavassa. Muista purkaa tilaus `return`-funktion avulla.

## 7. Automaattinen scrollaus ja ulkoasu (5 p)

* Scrollaa viestilista aina alareunaan, kun uusi viesti tulee, käyttämällä `useEffect` ja `ref`-elementtiä (kuten koodissa).
* Hyödynnä DaisyUI:n `chat-start` ja `chat-end` -luokkia viestien sijoittelussa oikealle/vasemmalle.

## 8. Käyttäjän tunnistaminen ja aikaleimat (5 p)

* Aseta oma `username`-tila, joko kovakoodattuna (kuten esimerkissä) tai kerää nimi käyttäjältä `prompt`-funktiolla.
* Näytä viestin aika selkeässä formaatissa: `new Date(message.created_at).toLocaleString('fi')` sekä loader-dataan että paikallisesti luotuihin viesteihin.

## 9. Syöttökentän validointi ja virheenkäsittely (5 p)

* Poista **Send**-painike käytöstä, jos syöttökenttä on tyhjä tai pelkkää tyhjää.
* Näytä virheilmoitus (esim. alert/toast), jos Supabase palauttaa virheen viestin lisäyksessä tai haussa. Käsittele myös reaaliaikaisen tilauksen mahdolliset virheet.

## 10. Koodin laatu ja linttaus (5 p)

* Aja `npm run lint` ja korjaa ESLintin ilmoittamat ongelmat. Alustapohja sisältää ESLintin valmiiksi.
* Käytä kuvaavia muuttujanimiä ja eriytä toistuva koodi omiksi komponenteiksi, jos se selkeyttää.
* Varmista, että listaelementeillä on vakaa `key`-prop (esim. viestin id).

## Bonus: React Routerin jatko (10 p, valinnainen)

React Router 7 tarjoaa paljon ominaisuuksia file route ‑pohjaisen reitityksen lisäksi. Saat lisäpisteitä tutkimalla ja toteuttamalla yhtä tai useampaa seuraavista:

* **Nested routes** - tee `/chat/settings`-sivu, jota voit käyttää chat-sovelluksen asetuksien säätämisessä, ja sisällytä se `chat.tsx`-moduulin sisäkkäiseksi reitiksi.
* **Loader ja action** - lisää `action`-funktio chat-reitille, joka lähettää viestin (ns. "mutation") lomakkeella. Lue lisää data loadingista React Routerin dokumentaatiosta.
* **Lazy route splitting** - kokeile `flatRoutes()`-funktiota tai `@react-router/fs-routes`-kirjastoa auttamaan reittien automaattista määrittelyä.

---

## Lähdekooditiedostot

### app/routes.ts

```ts
import { type RouteConfig, index, route } from "@react-router/dev/routes";

// Määrittele tiedostopohjaiset reitit.
// index() asettaa "home" polun etusivuksi.
// route("chat", "routes/chat.tsx") osoittaa chat-moduulin.
export default [
  index("routes/home.tsx"),
  route("chat", "routes/chat.tsx"),
] satisfies RouteConfig;
```

### app/services/supabase.ts

```ts
import { createClient } from "@supabase/supabase-js";

// Supabase-asiakkaan luonti.
// Dokumentaatio: https://supabase.com/docs/reference/javascript/initialize
const supabaseUrl = import.meta.env.VITE_SUPABASE_URL;
const supabaseKey = import.meta.env.VITE_SUPABASE_ANON_KEY;

const supabase = createClient(supabaseUrl, supabaseKey);

export default supabase;
```

### app/routes/chat.tsx

```tsx
import { useEffect, useRef, useState } from "react";
import { useLoaderData } from "react-router";
import type { Route } from "./+types/chat";
import supabase from "../services/supabase";

/*
  Loader: hakee viestit Supabase-taulusta palvelinpuolella.
  Lisätietoja loader-funktioista: https://reactrouter.com/start/framework/data-loading
*/
export async function loader({}: Route.LoaderArgs) {
  const { data, error } = await supabase
    .from("messages")
    .select("*")
    .order("created_at", { ascending: true });

  if (error) {
    throw new Error(error.message);
  }

  return { messages: data };
}

/*
  Chat-komponentti: käyttää loaderDataa, hallitsee tilaa ja
  kuuntelee Supabase realtime-rajapintaa. Käyttää DaisyUI-luokkia
  viestien asetteluun.
*/
export default function Component({
  loaderData,
}: Route.ComponentProps) {
  const [messages, setMessages] = useState(loaderData.messages);
  const [content, setContent] = useState("");
  const scrollRef = useRef<HTMLDivElement>(null);
  const username = "Testaaja"; // TODO: kysy käyttäjältä nimi

  async function addMessage() {
    if (!content.trim()) return;
    const { data, error } = await supabase
      .from("messages")
      .insert([{ username, content }])
      .select();
    if (error || !data) {
      alert("Virhe viestin lähetyksessä");
      return;
    }
    setMessages((curr) => [...curr, data[0]]);
    setContent("");
  }

  useEffect(() => {
    const channel = supabase
      .channel("chat-room")
      .on(
        "postgres_changes",
        { event: "INSERT", schema: "public", table: "messages" },
        (payload) => {
          // Vältä duplikaatit omille viesteille
          if (payload.new.username !== username) {
            setMessages((curr) => [...curr, payload.new]);
          }
        },
      )
      .subscribe();

    return () => {
      supabase.removeChannel(channel);
    };
  }, []);

  useEffect(() => {
    if (scrollRef.current) {
      scrollRef.current.scroll({
        top: scrollRef.current.scrollHeight,
        behavior: "smooth",
      });
    }
  }, [messages]);

  return (
    <main className="flex flex-col items-center gap-4 p-4">
      <h1 className="text-2xl font-bold">Chat</h1>
      <div
        ref={scrollRef}
        className="h-[40dvh] w-full max-w-[600px] overflow-y-auto border rounded p-2 space-y-2"
      >
        {messages.map((message: any) => {
          const isOwn = message.username === username;
          const timestamp = new Date(
            message.created_at ?? message.ts,
          ).toLocaleString("fi");
          return (
            <div
              key={message.id ?? message.ts}
              className={`chat ${isOwn ? "chat-end" : "chat-start"}`}
            >
              <div className="chat-header">
                {message.username}
                <time className="text-xs opacity-50 ml-2">
                  {timestamp}
                </time>
              </div>
              <div className="chat-bubble">{message.content}</div>
            </div>
          );
        })}
      </div>
      <textarea
        className="textarea w-full max-w-[600px]"
        placeholder="Kirjoita viesti"
        value={content}
        onChange={(e) => setContent(e.target.value)}
      ></textarea>
      <div className="flex w-full max-w-[600px] justify-end">
        <button
          onClick={addMessage}
          className="btn btn-primary"
          disabled={!content.trim()}
        >
          Send
        </button>
      </div>
    </main>
  );
}
```

