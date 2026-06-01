# Distributed Chord Protocol

O să mă iau să descriu funcționalitatea temei începând de la main, intrând prin toate ramificațiile acesteia în alte funcții și modul în care funcționează codul și cum am făcut implementarea

## 1. Inițializarea Programului (Main)
În main totul începe prin inițializarea programului:
* **Init**: Familiarizează programul nostru cu faptul că este rulat pe mai multe procese.
* **Comunicare**: Îi comunicăm care e rank-ul procesului curent și câte procese avem în total în cadrul programului nostru.
* **Citire**: După ce am comunicat programului datele de mai sus, citim din fișierul cu același rang pe care îl are și procesul curent și alocăm memoria în funcție de nr de lookups citit.

## 2. Construcția Topologiei (MPI & Finger Table)
* **MPIAllgather**: Folosim MPIAllgather ca fiecare proces să afle id-ul celorlalte procese. Practic procesele fac schimb între ele de ID-uri.
* **Mapare**: Construim pentru fiecare proces map-ul de id-uri, global ring-ul și finger table-ul (cu ajutorul funcției de la TOD01).
* **TODO 01 (build_finger_table)**: Funcția `build_finger_table` trece prin cele 4 noduri responsabile și setează valoarea nodului curent și al celui succesor.
* **find_succesor_simple**: Trece prin toate nodurile, în ordine folosind `sorted_ids`. Când a găsit un indice egal sau mai mare cu cheia căreia vrem să îi asignăm un nod responsabil, returnăm valoarea; dacă nu găsim, e clar că se află în intervalul de la ultimul și primul indice, deci returnăm indicele de pe prima poziție ca nod răspunzător/succesor.
* **Barieră**: După care vom aștepta până când toate procesele noastre pe care le avem inițializate ajung în punctul acesta.

## 3. Protocolul de Lookup

### TODO 04: Inițierea cererii
Construim mesajul de lookup pentru prima cheie din listă. Mesajul îl trimitem procesului curent, pe care îl vom decoda în todo 5. La pasul urmator din main, am setat câmpurile conform comentariilor din temă.

### TODO 05: Procesarea mesajelor
Aici prelucrăm toate mesajele primite, în funcție de tag, câte request-uri avem, și câte procese au terminat execuția.
* **Tag DONE**: Dacă numărul de lookup-uri este 0, vom trimite la toate celelalte procese mesajul done, pentru a le înștiința că unul dintre procese tocmai s-a terminat.
* **Contorizare**: Fiecare proces în parte va aștepta să primească done de la celelalte procese; am luat un contor, `done_received`, care contorizează câte mesaje done au fost primite de fiecare proces în parte.
* **Finalizare**: Când ajungem la un număr de mesaje done egal cu `world_size`, înseamnă că toate procesele și-au terminat execuția și singurul lucru care ne rămâne de făcut este să așteptăm până când se epuizează toate lookup-urile rămase, asta dacă nu cumva s-au epuizat deja.
 
### TODO 03: handle_lookup_request
Se așteaptă după mesaje și le procesăm în funcție de tag-ul lor. Dacă avem REQUEST, vom folosi funcția de la TOD03:
* Funcția primește o referință la un mesaj, pe care îl actualizăm cu `self.id`.
* Dacă cheia mesajului este chiar id-ul curent, nodul responsabil va fi chiar cel curent, altfel verificăm dacă cheia se află în interval.
* Dacă se află, nodul responsabil devine succesorul nodului curent. Actualizăm path-ul din mesaj, adăugăm nodul responsabil, incrementăm lungimea path-ului și trimitem procesului care a inițiat request-ul nodul responsabil în reply.
* Altfel, înseamnă că trebuie să continuăm căutarea pentru nodul responsabil și vom lua următorul nod folosind funcția `closest_preceding_function` de la TODO 2.

### TODO 02: closest_preceding_function
Parcurgem intrările din finger table de la cel mai mare la cel mai mic. Dacă avem finger-ul în interval, înseamnă că am găsit nodul căutat. Dacă în urma căutărilor se constată că niciun finger nu este cel dorit, întoarcem succesorul.

## 4. Gestionarea Răspunsurilor și Ieșirea
* **REPLY**: Dacă avem REPLY, afișăm calea lookup-ului și cresc index-ul curent. Dacă nu am ajuns la ultimul index, trimit lookup-ul următor.
* **Epuizare**: E posibil să nu mai am lookup-uri, deci dacă `idx_curent` ajunge la final, am terminat și acest proces și putem trimite `tag_done` la procesele rămase.
* **Monitorizare**: Dacă avem tag DONE incrementăm numărul de procese care și-au terminat treaba.
* **MPI_Finalize**: Apelăm MPI_Finalize pentru a marca faptul că ieșim din zona de procesare în paralel și programul își termină execuția.
