% Seguridad en aplicaciones Web Java
% (manuel.freire@fdi.ucm.es)
% 2019.03.11

# Introducción

## Objetivo

> Seguridad en aplicaciones Web Java

## Amenazas

* Denegación de servicio
* Destrucción de datos de usuarios
* Acceso a datos de usuarios
* Suplantación de identidad
* Acceso ó control externo de la aplicación
* Acceso ó control externo del servidor

## Vectores de ataque

* Externos a la aplicación: _puedes mitigar su impacto_
    + acceso físico al servidor
    + vulnerabilidad remota
    + vulnerabilidad local
* Internos a la aplicación: _puedes evitarlos completamente_
    + inyección de SQL (contra la BD)
    + inyección de HTML/JS (contra clientes)
    + inyección de ficheros (contra datos/programa)
    + datos incorrectos (contra integridad/validez)

# Vectores externos

## ¿Tu servidor está comprometido?

* Defensa en profundidad
    + primero, que no entren
    + si entran, que no se lo lleven _todo_

* Evita exponer datos sensibles
    + [contraseñas: ~7700 M cuentas comprometidas](https://haveibeenpwned.com/)
    + tarjetas / información de pago
    + otra información personal delicada

## Contraseñas: qué NO hacer

* NUNCA guardes contraseñas en _texto claro_
    + no en tu BD
    + no en tus logs (incluso si son fallidas)
* NUNCA envies contraseñas en _texto claro_
    + para 'resetear' acceso, usa tokens aleatorios con expiración (_nonce_)

## Contraseñas: porqué NO hacerlo

* Usuarios a menudo
    + reutilizan contraseñas en muchos sitios
    + usan un "esquema de contraseña" más o menos atacable (`cebra18`, `leon19`, ... ) para inventar nuevas contraseñas. 

* Con contraseñas en claro, atacante tiene vía libre
    + contra tu sitio, como cualquier usuario (por ejemplo, admin)
    + contra sitios externos, contra usuarios que reutilizan ó usan esquema

## qué NO hacer (x2)

* NUNCA guardes contraseñas en _texto claro_
    + no en tu BD
    + no en tus logs (incluso si son fallidas)
* NUNCA envies contraseñas en _texto claro_
    + para 'resetear' acceso, usa tokens aleatorios con expiración (_nonce_)

## Contraseñas: qué SI hacer

* Convierte tus contraseñas a un **formato modificado** que
    + permite verificar el original corresponde a la modificación
    + no permite obtener el original a partir de la modificación
    + no permite entrar con la modificación

* Receta para un buen **formato modificado**
    + sal
    + clave
    + pimienta (opcional)
    + y todo ello pasado por una buena batidora de bits (**hash**)

## Contraseñas: Hashes

* Dada una cadena de bits _b_, h(b) (un hash) debe cumplir
    + consistente: $b_1 = b_2 \implies h(b_1) = h(b_2)$
    + longitud constante: $\forall b, |h(b)| = cte$
* Como hay más claves que hashes, habrá _colisiones_
    + colisión: $a \neq b \wedge h(a) = h(b)$ 
    
* Para ser criptográficamente útil:
    + dado $h(b_1)$, debe ser difícil encontrar $b_2 | h(b_1) = h(b_2)$ (pre-imagen)
    + dado $b_1$, debe ser difícil encontrar $b_2 | h(b_1) = h(b_2)$ (segunda pre-imagen)
    + y, en general, debe ser difícil encontrar colisiones (resistencia fuerte)

- - - 

Notación 

$h$($a$ **concatenado_con** $b$) = $h(a || b)$ 

- - - 

## Contraseñas: Condimento

* Contra un hash puro, almacenando $h(pass)$
    + hash de gran diccionario: ¿alguna coincide?
    + extensión del diccionario: *rainbow tables*
    + miro lista de hashes: ¿alguna coincide?

(referencias: [XKCD](http://xkcd.com/1286/), [crucigrama al respecto](http://zed0.co.uk/crossword/) ) 

- - -

* Contra un hash "salado", almacenando $h(sal || pass)$ y $sal$
    + $h(s_1 || pass_1) = h(s_2 || pass_2) \not \Rightarrow pass_1 = pass_2 \quad$ (en general; y además, las colisiones son infrecuentes)
    + $h(s_1 || pass_1) \neq h(s_2 || pass_2) \not \Rightarrow pass_1 \neq pass_2 \quad$ (en general)
    + $\leadsto$ para encontrar contraseñas, necesitas atacarlas **cada una por separado**, con su sal - en lugar de todas a la vez y reaprovechando trabajo previo

## Mejores prácticas 

* No escribas código de gestión de contraseñas.
    + La criptografía es difícil
    + Mejor dejársela a los tipos que la saben y que van a actualizarla cuando se rompa
    + Coste de fuerza bruta siempre disminuye, nunca va a aumentar
* [Bcrypt](https://en.wikipedia.org/wiki/Bcrypt)
    + Multironda: aumenta coste de ataques (y de uso normal, pero es asimétrico: le "duele" mucho más al atacante). 
    + Distintos algorimos: empezó con `md5`, ahora se usa sobre todo con `blowfish`
    + Autodescriptivo: resultado de aplicarlo incluye información sobre algoritmo, rondas, y sal
    + Por defecto, Spring Security lo usa, con $2^{10}$ rondas: `$2a$10$`
* Alternativas (entre [muchas soportadas por Spring Security](https://docs.spring.io/spring-security/site/docs/current/api/org/springframework/security/crypto/factory/PasswordEncoderFactories.html))
    + Argon2
    + PBKDF2
    + Scrypt

## Tarjetas de crédito

* Peores que las contraseñas: 16 dígitos
    * 6 del banco (adivinables)
    * 1 redundante (se calcula a partir de los otros)
    * 4 últimos usados para resolver incidencias (semipúblicos, tendrías que guardarlos 'en claro')
    * adivinar 5 restantes por fuerza bruta es fácil
* Evita gestionarlas
    * Usa las APIs de tu proveedor
    * Que carguen _otros_ con las responsabilidades asociadas ([PCI](https://www.pcisecuritystandards.org/))

# Vectores internos

## Inyección SQL

> Texto del usuario que se <br>
>   incluye en una consulta SQL, <br>
> pero se interpreta como SQL <br>
>   en lugar de texto

. . .

Introducing: little [Bobby tables](http://bobby-tables.com/java.html)

- - - 

~~~{.java}
    login = "Robert"; // del formulario
    sql = "SELECT * FROM Student WHERE login='" + login + "';";
~~~

. . .

~~~{.sql}   
    SELECT * FROM Student WHERE login='Robert'
~~~

. . . 

---

~~~{.java}
    login = "Robert'); DROP TABLE Students;--"; // del formulario
    sql = "SELECT * FROM Student WHERE login='" + login + "';";
~~~

. . .

~~~{.sql}
    SELECT * FROM Student WHERE login='Robert'); DROP TABLE Students;--';
~~~

## Evitando inyección SQL

* Usa consultas preparadas
* No construyas consultas por concatenación
    + porque para evitar inyecciones tendrías que limpiar los argumentos...
* NO intentes 'limpiar' a mano
    + _no sabes suficiente SQL_
    + _ni conoces las variantes que usa cada BD_

## Inyección de HTML/JS

> Texto del usuario que se <br>
>   incluye en una página web, <br>
> pero se interpreta como HTML/JS<br>
>   en lugar de texto

. . .

Introducing: [alert(1) to win](http://escape.alf.nu/)

- - -

~~~{.java}
// en el controlador
login = "Berto"; // del formulario
model.addAttribute("login", login);
~~~
~~~{.html}
<!-- en la vista;
    (nunca useis 'utext' con datos controlados por el usuario) -->
<h1>Bienvenido, <span th:utext="${login}">Alguien</span></h1>
~~~

. . .

~~~{.html}
<h1>Bienvenido, <span>Berto</span></h1>
~~~

. . . 

---

~~~{.java}
// en el controlador
login = "<script>alert(1)</script>"; // del formulario
model.addAttribute("login", login);
~~~
~~~{.html}
<!-- en la vista;
    (nunca useis 'utext' con datos controlados por el usuario) -->
<h1>Bienvenido, <span th:utext="${login}">Alguien</span></h1>
~~~

. . .

~~~{.html}
<h1>Bienvenido, <span><script>alert(1)</script></span></h1>
~~~

---

~~~{.html}
<h1>Bienvenido, <span><script>alert(document.cookie)
</script></span></h1>
~~~

## Evitando inyección HTML/JS

* Valida, valida y valida. Y luego, escapa.

* Escapado: depende del contexto
    + dentro de un elemento HTML: thymeleaf os ayuda; usa `text`
    + dentro de un atributo de HTML: thymeleaf os ayuda: lo escapa automáticamente. 
    + dentro de JS: configura bien tu serialización (la veremos el próximo día).

## Incluyendo JS dinámico en tu HTML

~~~{.java}
model.addAttribute("message", "1 + Math.sqrt(2)");
~~~

~~~{.javascript}
// dentro de un <script> en un fichero html
var message =/*[[${message}]]*/ 'defaultanyvalue';
//            |              |
//            +--------------+-- sin ' ' entre comentario y corchetes
~~~

## Vectores internos: ficheros

* Desconfía de los ficheros que te suban
    + de su nombre y/o ruta: descártalo
    + de su contenido: no te fíes de lo que te dicen
* Usa un esquema de nombrado que garantice que no hay colisiones
    + recomendado: _base_/_tabla_/_id-en-tabla_

(En J2EE es mucho más complicado conseguir que el servidor ejecute archivos externos que en PHP)
    
## Bajando ficheros en la aplicación de ejemplo

~~~{.java}
@GetMapping(value="/{id}/photo", produces = MediaType.IMAGE_JPEG_VALUE)
public StreamingResponseBody getPhoto(@PathVariable long id, 
        Model model) throws IOException {
    File f = localData.getFile("user", ""+id);
    InputStream in;
    if (f.exists()) {
        in = new BufferedInputStream(new FileInputStream(f));
    } else {
        in = new BufferedInputStream(getClass().getClassLoader()
            .getResourceAsStream("static/img/unknown-user.jpg"));
    }
    return new StreamingResponseBody() {
        @Override
        public void writeTo(OutputStream os) throws IOException {
            FileCopyUtils.copy(in, os);
        }
    };				
}
~~~

## Subiendo ficheros en la aplicación de ejemplo

~~~{.java}
@PostMapping("/{id}/photo")
@ResponseBody
public String postPhoto(@RequestParam("photo") MultipartFile photo,
        @PathVariable("id") String id){
    File f = localData.getFile("user", id);
    if (!photo.isEmpty()) {
        try (BufferedOutputStream stream = 
                new BufferedOutputStream(new FileOutputStream(f))) {
            byte[] bytes = photo.getBytes();
            stream.write(bytes);                
        } catch (Exception e) {
            return "Error uploading " + id + " => " + e.getMessage();
        }
    } else {
        return "You failed to upload a photo for " + id + ": empty?";
    }
    return "You successfully uploaded " + id + " into " 
        + f.getAbsolutePath() + "!";
}
~~~

## Validando que son lo que dicen ser

~~~{.java}
boolean isImage(File f) {
    try {
        BufferedImage img = ImageIO.read(new File("strawberry.jpg"));
    } catch (IOException e) {
        return false;
    }
    return true;
}
~~~

* Nota: esto sólo garantiza que son imágenes. Pueden ser, además, más cosas:
[en este caso, las obras de Shakespeare y su retrato](https://twitter.com/David3141593/status/1057042085029822464)

## Vectores internos: datos incorrectos
    
* Nunca asumas que lo que llega es lo que esperas recibir
    + Es posible escribir URLs sin hacer click en enlaces
    + Es posible modificar parámetros GET editando la URL
    + Es posible modificar parámetros POST editando JS ó usando utilidades del navegador

- - - 

~~~~
    /admin?op=borraUsuario&login=paco
~~~~

. . .

más te vale comprobar si el que pide esto tiene permiso para borrar a 'paco'

~~~~
    /carrito?producto=123&cantidad=2&precio=36
~~~~

. . .

el precio nunca debería formar parte de esta consulta: ¡pueden cambiarlo!

## Datos incorrectos

* VALIDA tus formularios en el CLIENTE
    + esto evita enfadar a tus clientes por tener que re-introducirlos
    + esto evita algunas peticiones malas en el servidor
* VALIDA tus formularios en el SERVIDOR (y los permisos de quien los envía) 
    + esto evita corromper tus datos
    + esto evita que se salten tus permisos

# Fin

## ¿?

¡No te quedes con preguntas!

------

<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/"><img alt="Creative Commons License" style="border-width:0" src="img/88x31.png" /></a><br />This work is licensed under a <a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/">Creative Commons Attribution-ShareAlike 4.0 International License</a>.
