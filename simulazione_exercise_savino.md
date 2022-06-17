# Simulazione Exercise Savino

Domande forniteci dal prof Savino tramite piattaforma exercise come esempio di contenuto dell'esame prima del primo appello.

### Domanda 1
- **Q1**: "Si scriva un programma in cui una funzione generica (_generateVector_ da scrivere a sua volta) generi un vettore accettando in ingresso il numero di elementi, facendo in modo che il programma principale ne usi il risultato per stamparlo a video, minimizzando il contributo di memoria usata."
- **Risposta**:
```Rust
fn generateVector<T : Default + Clone>(n: usize) -> Vec<T> {
    vec![T::default(); n]
}

fn main() {
    let strings: Vec<String> = generate_vector(10);
    let integers: Vec<i32> = generate_vector(10);
    let floats: Vec<f64> = generate_vector(10);
    println!("{:?}", strings);    // ["", "", "", "", "", "", "", "", "", ""]
    println!("{:?}", integers);    // [0, 0, 0, 0, 0, 0, 0, 0, 0, 0]
    println!("{:?}", floats);    // [0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0]
}
```

### Domanda 2
- **Q2**: "a) Si scriva una struttura con implementazioni o Tratti (o classe generica se C++) (chiamata _MinEvaluator_) che consenta di calcolare il minimo tra due vettori di interi, dove il minimo si intende il vettore con la somma degli elementi più piccola.

    b) E' possibile crearne una versione in cui invece del minimo si possa anche invertire il confronto senza che ne esistano due versioni diverse? Se sì, indicare la versione _Evaluator_ che lo consenta.

    c) Infine si scriva, basandosi sulla versione Evaluator, una versione in cui la valutazione sia essere eseguita in parallelo." <- il prof ha perso la capacità di usare l'italiano qui 💩
- **Risposta**:
```Rust
//
//  ---------------------------- a) ---------------------------- //
//
#[derive(Debug)]
struct MinEvaluator {
    v1: Vec<i32>,
    v2: Vec<i32>
}

impl MinEvaluator {

    pub fn new(v1: Vec<i32>, v2: Vec<i32>) -> MinEvaluator {
        MinEvaluator {
            v1,
            v2
        }
    }

    pub fn get_min(&self) -> Vec<i32> {
        let sum1 = self.v1.iter().fold(0,|tot, e| tot + e);
        let sum2 = self.v2.iter().fold(0,|tot, e| tot + e);
        match sum1 < sum2 {
            true => self.v1.clone(),
            false => self.v2.clone()
        }
    }
}


fn main() {
    let v1 = vec![1,2,3,4];
    let v2 = vec![5,6,7,8];
    let minev = MinEvaluator::new(v1, v2);
    println!("{:?}", minev);    // MinEvaluator { v1: [1, 2, 3, 4], v2: [5, 6, 7, 8] }
    println!("the min vector is {:?}", minev.get_min() ) // [1, 2, 3, 4]

}

//
//  ---------------------------- b) ---------------------------- //
//
#[derive(PartialEq)]
enum Order {
    Max,
    Min
}

#[derive(Debug)]
struct Evaluator {
    v1: Vec<i32>,
    v2: Vec<i32>
}

impl Evaluator {

    pub fn new(v1: Vec<i32>, v2: Vec<i32>) -> Evaluator {
        Evaluator {
            v1,
            v2
        }
    }

    pub fn get(&self, order: Order) -> Vec<i32> {
        let sum1 = self.v1.iter().fold(0,|tot, e| tot + e);
        let sum2 = self.v2.iter().fold(0,|tot, e| tot + e);
        match !((sum1 < sum2) ^ (order == Order::Min)) {
            true => self.v1.clone(),
            false => self.v2.clone()
        }
    }
}

fn main() {
    let v1 = vec![1,2,3,4];
    let v2 = vec![5,6,7,8];
    let minev = Evaluator::new(v1, v2);
    println!("{:?}", minev);
    println!("the min vector is {:?}", minev.get(Order::Min) ); // [1, 2, 3, 4]
    println!("the max vector is {:?}", minev.get(Order::Max) ); // [5, 6, 7, 8]

}

//
//  ---------------------------- c) si scriva, basandosi sulla versione Evaluator, una versione in cui la valutazione sia essere eseguita in parallelo. ---------------------------- //
//

/*
Per usare num_cpus aggiungi in cargo.toml:

[dependencies]
num_cpus = "1.0"
*/

use num_cpus;
use std::thread;
use std::thread::{JoinHandle, sleep};
use std::time::Duration;


#[derive(PartialEq)]
enum Order {
    Max,
    Min
}

#[derive(Debug)]
struct Evaluator {
    v1: Vec<i32>,
    v2: Vec<i32>
}

impl Evaluator {

    pub fn new(v1: Vec<i32>, v2: Vec<i32>) -> Evaluator {
        Evaluator {
            v1,
            v2
        }
    }

    pub fn sum_concurrently(len: usize, v: &Vec<i32>) -> i32 {
        println!("len: {}, v: {:?}",len, v);
        let cpus_physical = num_cpus::get_physical();
        let threads_tot = match cpus_physical <= v.len() {
            true => cpus_physical,
            false => v.len()
        };

        let jh_vec: Vec<JoinHandle<i32>> = (0..threads_tot).map(|tid| {
            let v_thread = match tid == threads_tot-1 {
                false => v[(len/threads_tot*tid)..(len/threads_tot*(tid+1))].to_vec(),
                true => v[(len/threads_tot*tid)..].to_vec(),
            };
            thread::spawn(move || {
                let sum = v_thread.iter().fold(0,|tot, e| tot + e);
                println!("Thread {} fatto", tid);
                sum
            })
        }).collect();

        println!("Attendo la terminazione dei thread secondari");
        let total = jh_vec.into_iter().map(|h|{
            let res = h.join();
            match res {
                Ok(val) => val,
                Err(e) => {
                    println!("Error({:?}", e);
                    0
                }
            }
        }).fold(0,|tot, e| tot + e);
        total
    }

    pub fn get(&self, order: Order) -> Vec<i32> {

        println!("---------------THREADS TIME STARTS HERE-----------------------------");
        println!("***THREADS FOR V1***");
        let sum1 = Self::sum_concurrently(self.v1.len(), &self.v1);
        println!("sum 1 = {}",sum1);
        println!("***THREADS FOR V2***");
        let sum2 = Self::sum_concurrently(self.v2.len(), &self.v2);
        println!("sum 2 = {}",sum2);
        println!("---------------THREADS TIME ENDS HERE-----------------------------");

        match !((sum1 < sum2) ^ (order == Order::Min)) {
            true => self.v1.clone(),
            false => self.v2.clone()
        }
    }
}

fn main() {
    let v1 = vec![1,2,3,4,5,6,7,8,9];
    let v2 = vec![9,10,11,12,13,14,15,16,17];
    let minev = Evaluator::new(v1, v2);

    println!("{:?}", minev);
    println!("the min vector is {:?}", minev.get(Order::Min) );
    println!("the max vector is {:?}", minev.get(Order::Max) );

}
```

### Domanda 3
- **Q3**: "Si definisca il modello esecutivo delle condition variables. Si illustri con un esempio di codice le situazioni in cui sia necessario introdurre tale costrutto, sviluppando due funzioni thread"
- **Risposta**:
  Le condition variables sono strutture dati di sincronizzazione fra thread che permettono di attendere che si verifichi una condizione  (rappresentata da un espressione booleana) evitando il polling. In particolare la valutazione di questa condizione viene fatta dal thread bloccato solo dopo aver acquisito il mutex associato all condition variable in seguito alla notifica del cambiamento di stato della condizione da parte di un altro thread.
```Rust
fn main () {
    let pair = Arc::new((Mutex::new(false), Condvar::new()));
    let pair2 = Arc::clone(&pair);

// Inside of our lock, spawn a new thread, and then wait for it to start.
    thread::spawn(move|| {
        let (lock, cvar) = &*pair2;
        let mut started = lock.lock().unwrap();
        sleep(Duration::from_secs(2));
        println!("second thread: now i change the condition");
        *started = true;
        // We notify the condvar that the value has changed.
        cvar.notify_one();
        println!("second thread: cvar notified, thread finished");
    });

// Wait for the thread to start up.
    let (lock, cvar) = &*pair;
    let mut started = lock.lock().unwrap();
    println!("Main thread: entering in wait mode for cvar");
    while !*started {
        started = cvar.wait(started).unwrap();
    }
    println!("Main thread: cvar unlocked");
}
    
```


### Domanda 4
- **Q4**: "Si definisca  il concetto di smart pointer e l'insieme degli smart pointers disponibili nel linguaggio C++ e Rust. Si descriva,  usando un esempio, cosa avviene quando uno smart pointer è affetto dalla fine improvvisa del programma a seguito di un eccezione."
- **Risposta**:
  Gli smart pointer sono tipi che possono essere usati sintatticamente come puntatori nativi ma che hanno ulteriori caratteristiche come ad esempio:
    - garanzia di inizializzazione e rilascio
    - conteggio dei riferimenti
    - Accesso esclusivo con attesa
    - ecc...
  In C++ gli smart ponter disponibili sono:
    - `std::shared_ptr<T>`
    - `std::unique_ptr<T>`
    - `std::weak_ptr<T>`
  In Rust sono:
    - `Box<T>, Rc<T>, Arc<T>, Weak<T>`
    - `Cell<T>, RefCell<T>, Cow<T>, Mutex<T>, RwLock<T>`
In C++ un throw lancia una exception e risale lo stack fino a trovare un blocco catch che fa riferimento al suo tipo di errore. In rust viene invocata la macro panic! poichè non esiste il concetto di exception.
in entrambi i cai con l'unroll dello stack vengono eseguiti i distruttori degli smart pointer man mano che vengono rilasciati (?? non so cosa vuole sta domanda)


### Domanda 5
- **Q5**: "Si definiscano le caratteristiche esecutive di un processo, distinguendo un processo singolo ed uno in cui vi sia duplicazione (SO windows e Linux). Si definiscano inoltre i principi che legano il singolo processo ai thread in entrambe le situazioni (dalla prospettiva del SO linux)".
- **Risposta**:
  I processi sono unità base di esecuzione di un programma costituite da almeno un thread (il main thread).
  In windows i processi costituiscono entità separate che tra loro non hanno relazioni di dipendenza a differenza di quanto accadde in Linux dove sono organizzati secondo alberi di dipendenza processi padre -> processi figli.
  Quando si crea un nuovo processo su windows con la `CreateProcess(...)` si alloca un nuovo spazio di indirizzamento che viene poi inizializzato con l'immagine di un eseguibile e a cui viene avviato il thread primario, in Linux invece la `fork()` genera un nuovo spazio di indirizzamento identico a quello del processo in cui viene eseguita la fork e ogni pagina di memoria fisica condivisa fra i due processi viene etichettata come COW.
  L'approccio di Linux permette di creare processi molto più velocemente, inoltre si può sostituire l'attuale immagine di memoria dello spazio di indirizzamento utilizzando le funzioni `exec*([qui l'eseguibile con cui reinizializzare il processo])`.
  Per quanto riguarda Linux, l'uso della `fork()` potrebbe creare dei problemi nel caso di programmi concorrenti (con più thread) perchè il processo figlio verrà generato con un unico thread e le strutture di sinncronizzazione presenti nel padre possono trovarsi in stati incongruenti al mommento della `fork()`.
  Per gestire meglio la creazione di un processo viene messa dunque a disposizione del programmatore la pthread_atfork() che registra tre funzioni da invocare rispettivamente 1) appena prima della fork 2) appena dopo la fork nel processo padre 3) appena dopo la fork nel processo figlio.


### Domanda 6
- **Q6**: "Si spieghi il significato del contesto nelle lambda function, in particolare in relazione al caso della cattura."
- **Risposta**:
  Il contesto di una lambada function in rust è lo scope nel quale viene definita e le cose che vi si trovano. 
  Con "cattura" si intende la possibilità di fare riferimento alle variabili del contesto in cui viene definita la lambda all'interno del corpo della lambda stessa.
  Questo "riferimento" può essere un vero e proprio riferimento, una copia o l'acquisizione del possesso (tramite move) della variabile.
  In Rust di default le variabili in una lambda vengono catturate tramite riferimento di sola lettura.
  Se serve mutarle, la lambda deve essere dichiarata `mut`, se serve prenderne possesso, bisogna anteporre allo spazio dedicato ai parametri la parola chiave `move` 


