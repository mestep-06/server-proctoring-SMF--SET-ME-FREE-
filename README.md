# server-proctoring-SMF--SET-ME-FREE-

Un projecte que implementa un servidor de proctoring;

QUE ES UN SERVIDOR PROCTORING?
Un servidor de proctoring és implementat per a evitar copies i trampes a proves virtuals fetes a alumnes.


COM ESTA CREAT AQUEST PROJECTE I QUINES FUNCIONALITATS TÉ?
El projecte esta desenvolupat en diferents endpoints, que constan de les següents parts:

- ALUMNE: Fet amb python utilitzant les llibreries (sys, requests i PyQt5)

Pèrdua del focus:
Quan l’alumne intenta sortir de l’entorn restringit (per exemple, canviant de finestra, minimitzant l’aplicació o passant a una altra pantalla), l’eina detecta immediatament aquesta pèrdua de control i bloqueja la vista principal. En aquest moment, apareix una pantalla de bloqueig amb un missatge ben destacat i un camp per introduir un codi d’autorització.

Pantalla de bloqueig:
Mentre la pantalla de bloqueig està activa, l’alumne no pot tornar al contingut fins que un professor faciliti un codi vàlid. Cada vegada que es produeix aquesta pèrdua de focus, el sistema envia internament una notificació al servidor perquè quedi registrada la incidència.

El procés per restaurar l’accés requereix dos passos d’autorització:

- Codi OTP: Aquest és un codi d’un sol ús que genera el professor manualment i només dura 30 segons.
- ID del professor: Aquesta seria la identificació del professor; cada professor té una ID única, i s’utilitza com una mesura extra de seguretat.

Si totes dues dades coincideixen amb les que estan registrades al servidor, la pantalla de bloqueig desapareix i l’alumne recupera l’entorn de l’examen tal com estava i pot continuar realitzant la prova/examen.


- PROFESOR: Fet amb Node.js

A l’hora d’utilitzar l’aplicació del professor, es presenten diverses funcionalitats, el que coneixem com a CRUD (Create, Read, Delete, Update). A continuació, explicarem com funciona cada part del CRUD:

- C (Create) L’aplicació pot crear codis OTP (One Time Password); aquests codis s’utilitzen a l’aplicació de l’alumne per desbloquejar el bloqueig que es crea quan l’alumne canvia de pestanya.

- R (Read) L’aplicació pot llegir les sessions que hi ha creades en aquell moment. El professor introdueix el seu ID i l’aplicació llegeix les sessions obertes que estan relacionades amb el mòdul que
  gestiona (ex.: si tinc l’ID 1 i gestiono MP01, només podré veure les sessions que hi ha en aquest MP amb el meu ID).

- U (Update) L’aplicació actualitza els mòduls que imparteixen els professors. Introdueixes el teu ID de professor i el mòdul que vols assignar, i l’aplicació ho modifica a la taula.

- D (Delete) L’aplicació també elimina els mòduls que té un professor, per tal de poder assignar un nou mòdul al professor en cas que l’actualització de mòduls deixi de ser útil.


El professor, en crear una sessió amb l’opció de «bloqueig» activada, desplega al costat de l’alumnat un petit script que:

Detecta quan l’alumne canvia de finestra o pestanya
Cada vegada que el navegador perd el focus (esdeveniment *blur*), s’envia automàticament un senyal al servidor.

Marca la sessió com a «bloquejada»
En rebre aquest avís, el *back-end* canvia l’estat de la sessió de l’alumne d’«activa» a «bloquejada».
Mentre està bloquejada, l’alumne no pot continuar l’examen fins que es restauri el focus.

Notifica al professor
Quan el professor consulta la llista de les seves sessions, veu l’estat de cada alumne assignat al seu mòdul, quan prem el botó «Carregar sessions»:


- SERVER: Implementat dins de una maquina virtual ubuntu-server, els fitxers per a la conexió estan fets amb PHP:

La part del server compta amb varies parts, que son les següents:

  - BASE DE DADES:
    
    A la màquina servidor s’utilitza MySQL. L’esquema que fem servir és:

  **Taula professors**
    Camps principals: id, nom, cognoms, correu, contrasenya (hash), module_code (NULL o «MP01»–«MP12»), otp, otp_expira.
    Un professor pot tenir com a màxim un mòdul assignat. El camp module_code indica aquesta assignació.

  **Taula alumnes**
    Camps: id, nom, cognoms, correu, module_code.
    Es crea o es cerca per nom + cognoms + module_code cada vegada que un alumne inicia sessió.

  **Taula sessions**
      Camps: id, alumne_id, data_inici, data_final (NULL mentre està activa).
    Cada vegada que un alumne entra (get_or_create), s’insereix un nou registre amb data_inici = NOW(). En finalitzar la sessió s’elimina.

  **Taula events**
      Camps: id, session_id, alumne_id, event_tip (INICI, PERDUA_FOCUS, FI), timestamp.
      Permet fer auditoria de pèrdues de focus i marcar l’inici i el final de la sessió.

  Les relacions FK garanteixen la consistència en eliminar un alumne o una sessió: els esdeveniments associats s’eliminen automàticament.

  - Integració SERVER-ALUMNE:

    get_or_create_alumne (POST JSON):
      L’alumne introdueix nom + cognoms + mòdul.
      El servidor cerca o crea l’alumne i inicia una sessió (sessions.data_inici).
      Respon amb {"success":true,"alumne_id":…, "session_id":…}.

    log_event (POST JSON):
      Cada vegada que l’aplicació detecta pèrdua de focus, envia {action:"log_event", session_id, alumne_id, event_tip:"PERDUA_FOCUS"}.
      També s’utilitza per marcar INICI i FI.

    end_session (POST JSON):
      Quan l’alumne finalitza, s’elimina de sessions (i per cascada d’events) i d’alumnes.
      El servidor retorna {"success":true,"message":"session_and_alumne_deleted"}.

      Totes les respostes fan servir JSON i l’endpoint exposa CORS (Access-Control-Allow-Origin: *), de manera que el client Python pot cridar des de qualsevol host sense restriccions.


    - Integració SERVER-PROFESOR:
   
      La interfície de professor (HTML/CSS/JS) fa peticions *fetch()* a professor.php per gestionar mòduls, OTPs i obtenir l’estat de les sessions:

    **Gestió de mòdul**
      Assignar (assign_module): POST FormData amb professor_id i module_code. El servidor verifica que no hi hagi ja un mòdul i actualitza professors.module_code.
      Actualitzar (update_module): POST anàleg, però només permet el canvi si ja existia un mòdul.
      Eliminar (delete_module): GET amb professor_id. Verifica que hi hagi un mòdul assignat i el posa a NULL.

    **OTP**
      Generar (generate_otp): GET amb professor_id. Crea un codi de 6 dígits, l’emmagatzema a professors.otp i retorna {otp, valid_until}.
      Verificar (verify_otp): GET amb professor_id i otp. Comprova que coincideix i que no ha caducat, retorna {valid:true}.

    **Llistar sessions (list_sessions):**
      GET amb professor_id. El servidor llegeix quin mòdul té assignat el professor i retorna totes les sessions de sessions els alumnes de les quals pertanyen a aquest mòdul.
      Per a cada sessió afegeix un camp booleà *lost_focus* que s’obté comprovant a events si existeix alguna fila amb event_tip='PERDUA_FOCUS'.

      També en aquest endpoint s’exposen capçaleres CORS perquè el *front-end* pugui cridar sense problemes des de localhost:3000 o un altre domini.





