# Semana 1 — El scraper, explicado

## El plot en una frase

`Scrapper.ipynb` es un robot de una sola misión: aterriza en una página web, le arranca el HTML hasta dejar solo texto, y se lo pasa a `gpt-4o-mini` para que devuelva un resumen en Markdown. Lo probaron contra `https://wired.com` y funcionó.

Piénsalo como el episodio piloto de tu propio laboratorio. Pied Piper todavía no comprime el internet, pero ya tiene un prototipo en el garaje.

---

## El reparto

La primera celda arma el equipo:

```python
import os
import requests
from dotenv import load_dotenv
from bs4 import BeautifulSoup
from IPython.display import Markdown, display
from openai import OpenAI
```

| Pieza | Qué hace en esta misión |
|---|---|
| `os` | Lee variables del sistema, incluida la llave secreta |
| `dotenv` | Abre el fichero `.env` y mete esas variables en el ambiente |
| `requests` | La nave. Hace el viaje HTTP y trae la página cruda |
| `BeautifulSoup` | El científico de a bordo. Abre el HTML y separa carne de carcasa |
| `OpenAI` | El modelo. Lee el texto y escribe el resumen |
| `Markdown`, `display` | Están en la mochila. Esta versión del notebook nunca los usa |

`requests` es la dropship de *Alien*: llega, toca suelo y vuelve con una muestra. `BeautifulSoup` es el laboratorio de la Nostromo: abre el espécimen y tira lo que no sirve. `OpenAI` es el cerebro que, al final, te cuenta qué había ahí dentro.

---

## Paso 1. La llave de la puerta

```python
load_dotenv()
api_key = os.getenv('OPENAI_API_KEY')

if not api_key:
    print("No se encontro ninguna api ")
elif not api_key.startswith("sk-proj-"):
    print("Se encontró un aclave Api pero no es el fromato correcto")
elif api_key.strip() != api_key:
    print("Se encontro una api key pero parece tener espacios")
else:
    print("Se encontro la clave API y hasta ahora parece buena")
```

Esto es el escáner de la esclusa, estilo *Black Mirror*: antes de dejar pasar a nadie, mira la credencial.

1. `load_dotenv()` lee el `.env`. La clave vive ahí, fuera del notebook, para que no quede pegada en el código como un post-it en el monitor de Gilfoyle.
2. `os.getenv('OPENAI_API_KEY')` la saca del ambiente.
3. Tres chequeos, en orden:
   - ¿Existe?
   - ¿Empieza por `sk-proj-`? Ese prefijo es el de las claves de proyecto de OpenAI.
   - ¿Tiene espacios al inicio o al final? `strip()` los quitaría; si el resultado cambia, había espacios y la clave fallaría en silencio.

Cuando se corrió, imprimió: **«Se encontro la clave API y hasta ahora parece buena»**.

Detalle de ingeniero: este bloque solo avisa. Si la clave estuviera mal, el notebook sigue adelante igual. Y la variable `api_key` no se le pasa al cliente. El cliente la vuelve a leer solo del ambiente.

---

## Paso 2. Encender el cliente

```python
openai = OpenAI()
```

Una línea. Creas el walkie-talkie hacia la API. Como no le pasas la clave a mano, `OpenAI()` busca `OPENAI_API_KEY` por su cuenta. Por eso el paso anterior tenía que dejarla bien puesta.

Es el momento Rick: el portal ya está calibrado. Todavía no has viajado a ningún universo.

---

## Paso 3. La clase `Website`, el facehugger

Esta es la pieza central. Una clase es un molde. Cada vez que escribes `Website(una_url)` nace un objeto que ya hizo el trabajo sucio en el constructor.

```python
class Website:
    def __init__(self, url):
        self.url = url
        response = requests.get(url)
        soup = BeautifulSoup(response.content, 'html.parser')
        self.title = soup.title.string if soup.title else "No tiene título"
        for irrelevant in soup.body(["script", "style", "img", "input"]):
            irrelevant.decompose()
        self.text = soup.body.get_text(separator="\n", strip=True)
```

Qué pasa, en orden, cuando nace el bicho:

1. **Guarda la URL** en `self.url`.
2. **`requests.get(url)`** pide la página. El servidor contesta con bytes: el HTML completo, menús, anuncios, scripts y todo.
3. **`BeautifulSoup(..., 'html.parser')`** convierte esos bytes en un árbol. Cada etiqueta (`<div>`, `<p>`, `<a>`) es una rama que ya puedes recorrer.
4. **El título.** Si existe `<title>`, se queda con su texto. Si la página no tiene, pone `"No tiene título"`.
5. **La autopsia.** Dentro de `<body>` busca `script`, `style`, `img` e `input` y les hace `decompose()`: los borra del árbol. JavaScript, CSS, imágenes y cajas de formulario no le sirven a un modelo que solo lee texto. Es Ash en la mesa de disección: quédate con el organismo, tira el traje.
6. **`get_text(separator="\n", strip=True)`** aplasta lo que queda a texto plano. Cada bloque va en su línea y se le quitan espacios sobrantes. Eso queda en `self.text`.

Al terminar, el objeto tiene tres cosas: `url`, `title` y `text`.

Lo que este facehugger todavía no hace, y por eso el resumen de WIRED salió lleno de «Menu», «Search» y «Subscribe»:

- No borra `<nav>`, `<header>` ni `<footer>`. El menú sobrevive y se lo comes después en el prompt.
- No mira si el servidor contestó 200. Un 404 también se parsea.
- No pone timeout. Si el sitio se cuelga, la celda se queda esperando como la tripulación en hipersueño.
- No ejecuta JavaScript. Si la página arma el contenido en el navegador después de cargar, `requests` se trae el cascarón vacío. Muy *Black Mirror*: la interfaz que ves en Chrome no es el HTML que viaja en el primer request.

---

## Paso 4. Primera prueba de vuelo: WIRED

```python
pagina = Website("https://wired.com")
print(pagina.title)
print(pagina.text)
```

Aquí el molde se usa de verdad. Nace `pagina`, viaja a WIRED, y escupe título y texto.

El título que salió:

`WIRED - The Latest in Technology, Science, Culture and Business | WIRED`

El texto es la portada aplastada: titulares de data centers, el Muse de Meta, estafas de tarjeta, Silicon Valley, más un montón de «Menu», «Headphone», «Listen» y el footer de Condé Nast. La muestra está viva. También está sucia. Suficiente para el prototipo del garaje de Erlich Bachman: funciona, y se ve el cableado.

---

## Paso 5. La personalidad del modelo

```python
system_prompt = "Eres un asistente que analiza el contenido de un sitio web \
y proporciona un breve resumen, ignorando el texto que podría estar relacionado con la navegación. \
Responder en Mark con referencias muy geek y d eingeniero humor tipo rick and morty big bang theory etcdown."
```

En la API de chat hay dos voces. **System** es el director de la serie: le dice al modelo quién es y cómo debe hablar. **User**, más abajo, es el episodio concreto.

Este system prompt le pide tres cosas:

- Resume el sitio.
- Ignora el texto de navegación. Por eso el menú puede quedarse en el scrape: le encargas al modelo que lo filtre, en vez de filtrarlo tú en el HTML.
- Responde con humor de ingeniero, tipo *Rick and Morty* y *The Big Bang Theory*.

El prompt tiene typos (`Mark`, `d eingeniero`, `etcdown`). El modelo igual entendió «Markdown» y «humor geek». Los modelos perdonan el teclado a las 2 a.m. Un contrato con otro programa no lo perdonaría.

---

## Paso 6. El encargo del usuario

```python
def user_prompt_for(website):
    user_prompt = f"Estás viendo un sitio web titulado {website.title}"
    user_prompt += "\nEl contenido de este sitio web es el siguiente; \
    proporciona un breve resumen de este sitio web en formato Markdown. \
    Si incluye noticias, productos o anuncios, resúmelos también.\n\n"
    user_prompt += website.text
    return user_prompt
```

Esta función arma el mensaje humano. Tres bloques pegados:

1. El título, con un f-string: `f"... {website.title}"`.
2. La orden: resume en Markdown, y si hay noticias, productos o anuncios, mételos también.
3. El texto crudo que sacó `Website`.

La barra `\` al final de la línea solo dice «esta frase sigue abajo». No forma parte del texto que ve el modelo.

Al imprimirlo con `print(user_prompt_for(pagina))` sale exactamente eso: instrucciones y, debajo, toda la portada de WIRED. Es el sobre que vas a meter en el portal.

---

## Paso 7. Empaquetar el chat

```python
def messages_for(website):
    return [
        {"role": "system", "content": system_prompt},
        {"role": "user", "content": user_prompt_for(website)}
    ]
```

La API no acepta «un párrafo». Acepta una lista de turnos, cada uno con `role` y `content`.

- Turno 1, `system`: las reglas del personaje.
- Turno 2, `user`: título, orden y página entera.

Es el formato de *Rick and Morty* cuando Rick le grita instrucciones a Morty y después le tira el problema encima. Primero el carácter. Después la misión. Al llamar `messages_for(pagina)`, el notebook muestra esa lista de dos diccionarios, con el system prompt y el texto de WIRED ya dentro.

---

## Paso 8. `summarize`: los cuatro se convierten en uno

```python
def summarize(url):
    website = Website(url)
    response = openai.chat.completions.create(
        model = "gpt-4o-mini",
        messages = messages_for(website)
    )
    return response.choices[0].message.content
```

Esta función es el botón rojo. Le das una URL y ella encadena todo:

1. `Website(url)` aterriza, parsea y limpia.
2. `messages_for(website)` arma system + user.
3. `openai.chat.completions.create(...)` manda eso a **`gpt-4o-mini`**. Es el modelo chico y barato de la familia 4o: rápido, bueno para resumir, el intern de Hooli que sí entrega el ticket.
4. La respuesta trae una lista `choices`. Te quedas con la primera, `choices[0]`, y de ahí sacas `message.content`: el texto del resumen.

`summarize("https://wired.com")` devolvió un Markdown que describe WIRED como revista de tecnología, ciencia, cultura y negocios, con titulares (data centers, Muse de Meta, estafas retro, un agente de OpenAI en un servicio de salud de Australia) y chistes de *Rick and Morty* y *The Big Bang Theory*. El pipeline completo cerró: nave, laboratorio, prompt, modelo, texto de vuelta.

---

## Cómo se ve el viaje entero

```text
URL
  → requests.get          trae el HTML
  → BeautifulSoup         lo vuelve árbol
  → decompose             borra script, style, img, input
  → get_text              deja texto plano
  → system + user         arman el chat
  → gpt-4o-mini           escribe el resumen
  → choices[0].content    te lo devuelve
```

Una URL entra. Un resumen en Markdown sale. En medio hay una sonda que no entiende la página: solo la desarma. El que entiende es el modelo, y entiende lo que el prompt le deja ver.

Eso es lo elegante del notebook, y también su límite. El scraper es tonto a propósito, como el androide que solo obedece el protocolo. La inteligencia está del otro lado del cable, en `gpt-4o-mini`, condicionada por dos frases: ignora el menú, y cuéntalo como si Rick estuviera de mal humor en una daily de Silicon Valley.
