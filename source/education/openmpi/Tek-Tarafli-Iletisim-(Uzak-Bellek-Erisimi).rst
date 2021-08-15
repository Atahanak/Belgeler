
Tek Taraflı İletişim (Uzak Bellek Erişimi)
==========================================

MPI'daki ``MPI_Send`` ve ``MPI_Recv`` iletişim modeline artık aşinasınız. 
Bu model aynı zamanda iki taraflı iletişim olarak da adlandırılır ve iki süreç örtük olarak birbiriyle senkronize olur. 
Bu, birini aramak gibidir: iki tarafında aynı anda iletişim kurması gerekir. 
Elbette daha önce bahsedildiği gibi arabellek kullanımı ile basit, tek bir kere gerçeklenen
iletişimlerde, ya da tıkanmasız iletişim fonksiyonları ile bu senkronizasyon en aza indirgenebilir. 
Yine de bu model, veri aktarımı için her zaman en uygun model değildir. 
Çoğu zaman iki tarafında aynı anda senkronize olması gereksinimi verimsizliklere yol açar. 

Yukarıda bahsedilen nedenlerden dolayı, MPI, tek taraflı iletişim olarak da bilinen, özel bellek pencerelerinde sağlandığı 
sürece, işlemlerin diğer işlemlerdeki verilere erişebildiği, uzaktan bellek erişimi (RMA) için fonksiyonlar sunar.
Tek taraflı iletişim bir e-postaya benzer. Bir mesaj göndermek istediğinizde mesajınız arkadaşınızın gelen kutusunda kalacak, 
siz ise gönder düğmesine bastıktan sonra başka işlemler gerçekleştirebileceksiniz. 
Arkadaşınız ise boş zamanlarında yolladığınız e-postayı okuyacaktır.

Tek Taraflı İletişim Nasıl Çalışır?
-----------------------------------

İşlem 0, işlem 1'e bir veri göndermek istesin. İşlemler arasındaki her etkileşimin açık bir şekilde tanımlanması MPI için temeldir, 
bu nedenle basit bir değer ataması yeterli olmayacaktır. İlk olarak, hedef işlemde belleğin bir bölümünü, 
bu durumda işlem 1'i, işlem 0'ın işlemesi için görünür hale getirmeliyiz. Bu ayırdığımız belleği pencere olarak adlandırıyoruz.

İşlem 1'in belleğine bir pencere açıldığında, işlem 0 ona erişebilir ve onu değiştirebilir. İşlem 0, ``MPI_Put`` 
kullanarak yerel belleğindeki verileri işlem 1'in bellek penceresine koyabilir.

.. image:: /assets/openmpi-education/images/MPI_Win_put.png
   :target: /assets/openmpi-education/images/MPI_Win_put.png
   :alt: /assets/openmpi-education/images/MPI_Win_put.png

Bu görselde ayırdığımız pencereyi mavi eşkenar ile gösteriyoruz. İşlem 0 kendi belleğindeki *Y* değerini ``MPI_Put`` *RMA* 
fonksiyonunu kullanarak işlem 1'in penceresine depoluyor.

Başka bir farklı senaryoda, işlem 0 bellek penceresini bazı verilerle, örneğin *Y* ile, doldurmuş olsun: 
İletişimci'deki diğer herhangi bir işlem, örneğin işlem 1, artık ``MPI_Get`` *RMA* fonksiyonunu kullanarak 
bu verileri işlem 0'ın penceresinden alabilir. 

.. image:: /assets/openmpi-education/images/MPI_Win_get.png
   :target: /assets/openmpi-education/images/MPI_Win_get.png
   :alt: /assets/openmpi-education/images/MPI_Win_get.png

Yukarıda belirtildiği gibi bellek penceresi veya basitçe pencere terimiyle, uzak bellek erişimleri için ayrılmış, 
her işlem için yerel olan belleğe atıfta bulunuruz. Bir pencere nesnesi bunun yerine iletişimci'deki tüm süreçlerin 
pencerelerinin toplamıdır ve ``MPI_Win`` tipine sahiptir.

Bunlara ek olarak, pencerelerde gerçekleştirilen işlemler engellenmez yapıdadır. Tıkanmasız bu işlemler, programcının 
hesaplama ile iletişimi örtüştürmesine izin verirken, aynı zamanda açık senkronizasyon yükünü de beraberinde getirir.

.. image:: /assets/openmpi-education/images/MPI_senkron.png
   :target: /assets/openmpi-education/images/MPI_senkron.png
   :alt: /assets/openmpi-education/images/MPI_senkron.png

Yukarıda, MPI ile tek taraflı iletişim kullanan bir uygulamada pencere oluşturma, RMA fonksiyonlarına çağrılar atma ve 
senkronizasyon zaman çizelgesi gösterilmektedir. İletişimci'deki her işlemde ``MPI_Win`` nesnelerinin oluşturulması, 
*RMA* fonksiyonlarının yürütülmesine izin verir. Uygulamanın güvenliğini ve doğruluğunu sağlamak için, pencereye yapılan her erişim senkronize 
edilmelidir. Bellek penceresiyle herhangi bir etkileşimin, senkronizasyon rutinlerine yapılan çağrılarla korunması gerektiğini 
unutmayın. Senkronizasyon çağrıları arasındaki olaylar *dönem* olarak adlandırılır.

RMA Sürecinin Anatomisi
-----------------------

Pencereler
^^^^^^^^^^

MPI, bellek pencerelerinin oluşturulması için 4 **toplu** (gruptaki her işlem tarafından çalıştırılan) rutin sağlar:

* ``MPI_Win_allocate``: bellek ayırır ve tek taraflı iletişim için pencere nesnesini oluşturur.
* ``MPI_Win_create``: tek taraflı iletişim için daha önceden ayırılmış bellekten bir pencere objesi oluşturur.
* ``MPI_Win_allocate_shared``: tek taraflı iletişim ve paylaşımlı hafıza erişimi için pencere nesnesini oluşturur.
* ``MPI_Win_create_dynamic``: tek taraflı iletişim için pencere oluşturur. Bu pencere RMA operasyonları için kullanabilen bir bellekle ilişkilendirilebilir.

``MPI_Win`` tipi bir tanıtıcı, iletişimcideki'deki tüm sıralarda uzaktan işlemler için kullanılabilir hale getirilen belleği yönetir. ``MPI_Win_free`` ile kullanıldıktan sonra bellek pencereleri serbest bırakılmalıdır.

Yükleme / Saklama
^^^^^^^^^^^^^^^^^

Uzak pencerelerde veri yüklemek veya saklamak için bir köken ve bir hedef süreç belirleyebiliriz. İki taraflı iletişimden farklı olarak, kaynak işlem veri aktarımını, verilerin nereden geldiği ve nereye gideceğini, tam olarak belirtir. Bu amaç için üç ana MPI rutini grubu vardır:

* Koymak için: ``MPI_Put`` and ``MPI_Rput``
* Almak için: ``MPI_Put`` and ``MPI_Rget``
* Toplamak için: ``MPI_Accumulate``\ , ``MPI_Raccumulate`` ve varyasyonları.

Senkronizasyon
^^^^^^^^^^^^^^

Senkronizasyonun nasıl sağlanacağı, benimsenen tek taraflı iletişim paradigmasına bağlıdır:


#. Hem orijin hem de hedef süreçler senkronizasyonda rol oynuyorsa aktiftir. Bu paradigma paralel hesaplamanın mesaj iletme modelidir. ``MPI_Win_fence``\ , ``MPI_Win_start``\ , ``MPI_Win_complete``\ , ``MPI_Win_post``\ , and ``MPI_Win_wait`` aktif modelde kullanılmaktadır.
#. Kaynak işlemi veri aktarımını ve senkronizasyonunu düzenliyorsa pasiftir. Bu tip senkronizasyon paralel hesaplamanın paylaşılan bellek modeliyle yakından ilişkilidir: pencere, iletişimcideki paylaşılan bellektir ve her işlem, görünüşte birbirinden bağımsız olarak, üzerinde çalışabilir. ``MPI_Win_lock`` and ``MPI_Win_unlock`` pasif modelde kullanılmaktadır.

Pencere Yaratmak
^^^^^^^^^^^^^^^^

.. code-block:: c

   int MPI_Win_allocate(MPI_Aint size,
                        int disp_unit,
                        MPI_Info info,
                        MPI_Comm comm,
                        void *baseptr,
                        MPI_Win *win)

``size``: bayt olarak ayırılacak belleğin büyüklüğü.

``disp_uint``: yer değiştirme birimi, özellikle heterojen sistemler için gerekli, 1 olduğunda veri bayt olarak işlem görür.

``info``: MPI uygulamasına optimizasyon ipuçları sağlamak için kullanılabilecek bir bilgi nesnesi. ``MPI_INFO_NULL`` kullanmak her zaman doğrudur.

``comm``: programlar arası iletişimi sağlayan obje

``baseptr``: pencere için ayrılacak belliğin işaretçisi

``win``: *RMA* işlemlerinde kullanılacak pencere objesi

.. code-block:: c

   int MPI_Win_create(void *base,
                      MPI_Aint size,
                      int disp_unit,
                      MPI_Info info,
                      MPI_Comm comm,
                      MPI_Win *win)

``base``: pencere için ayrılacak belliğin işaretçisi

``MPI_Win_allocate`` hem bellek ayırma hem de pencere yaratma işlemlerini gerçekleştirirken, ``MPI_Win_create`` ise hali hazırda ayrılmış bir bellek üzerinde pencere yaratır.

RMA Aktarım Rutinleri
^^^^^^^^^^^^^^^^^^^^^

.. code-block:: c

   int MPI_Put(const void *origin_addr,
               int origin_count,
               MPI_Datatype origin_datatype,
               int target_rank,
               MPI_Aint target_disp,
               int target_count,
               MPI_Datatype target_datatype,
               MPI_Win win)

.. code-block:: c

   int MPI_Get(void *origin_addr,
               int origin_count,
               MPI_Datatype origin_datatype,
               int target_rank,
               MPI_Aint target_disp,
               int target_count,
               MPI_Datatype target_datatype,
               MPI_Win win)

Hem ``MPI_Put`` hem de ``MPI_Get`` tıkanmasız  değildir: 
senkronizasyon rutinlerine yapılan bir çağrı ile tamamlanırlar. İki işlev aynı argüman 
listesine sahiptir. ``MPI_Send`` ve ``MPI_Recv``\ 'e benzer şekilde, veriler adres, 
sayı ve veri tipi üçlüsü ile belirlenir. Ana işlemdeki veriler için bu üçlü şu şekildedir: ``origin_addr``\ , ``origin_count``\ , 
``origin_datatype``. Hedef işlemde, arabelleği yer değiştirme, sayım ve veri türü açısından tanımlarız 
(``target_disp``\ , ``target_count``\ , ``target_datatype``). Hedef süreçteki ara belleğin adresi, 
``MPI_Win`` nesnesinin temel adresi ve yer değiştirme birimi kullanılarak hesaplanır:

.. code-block:: c

   target_addr = win_base_addr + target_disp * disp_unit

.. code-block:: c

   int MPI_Accumulate(const void *origin_addr,
                      int origin_count,
                      MPI_Datatype origin_datatype,
                      int target_rank,
                      MPI_Aint target_disp,
                      int target_count,
                      MPI_Datatype target_datatype,
                      MPI_Op op,
                      MPI_Win win)

``MPI_Accumulate`` için bağımsız değişken listesi, hedef işlem üzerinde hangi indirgeme işleminin 
yürütüleceğini belirten ``MPI_Op`` tipindeki ``op`` parametresi dışında ``MPI_Put`` ile aynıdır. 
Bu rutin element bazında atomiktir: birden fazla süreçten erişimler belirli bir sırayla seri hale 
getirilir ve bu nedenle, hiçbir yarış koşulu (race condition) oluşamaz.  İndirgemeler yalnızca, işlem verilen veri 
türü için ilişkisel ve değişmeliyse belirleyicidir, bu yüzden dikkatli olmanız gerekir. 
Örneğin, ``MPI_SUM`` ve ``MPI_PROD``\ , kayan nokta (\ *floating point*\ ) sayıları için ne ilişkisel ne de değişmelidir!

Senkronizasyon Rutinleri
^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: c

   int MPI_Win_fence(int assert,
                     MPI_Win win)

``MPI_Win_fence`` bir pencereye dahil olan bütün işlemleri senkronize etmek için kullanılır.

.. code-block:: c

   int MPI_Win_lock(int lock_type,
                    int rank,
                    int assert,
                    MPI_Win win)

``lock_type``: ``MPI_LOCK_EXCLUSIVE`` veya ``MPI_LOCK_SHARED`` olabilir. 
``MPI_LOCK_EXCLUSIVE`` bir sıraya özgü bir kilit sağlarken, ``MPI_LOCK_SHARED`` ile bütün sıralar belleğe erişebilirler.

``rank``: penceresi kilitlenecek işlemin sırası.

``assert``: MPI kitaplığına optimizasyon ipuçları sağlamak için bu bağımsız değişkeni kullanırız. 
Bu argümanı ``0`` olarak atamak her zaman doğrudur.

``win``: pencere objesi

.. code-block:: c

   int MPI_Win_unlock(int rank,
                      MPI_Win win)

``MPI_Win_lock`` ve ``MPI_Win_unlock`` dağıtık bellekleri, yazılım seviyesinde, paylaşılmış bellekler gibi kullanmamıza olanak sağlar.

Aktif Senkronizasyon Örnek
^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: c

   #include <stdio.h>
   #include <stdlib.h>

   #include <mpi.h>

   int main(int argc, char *argv[]) {
     MPI_Init(&argc, &argv);

     MPI_Comm comm = MPI_COMM_WORLD;

     int size;
     MPI_Comm_size(comm, &size);
     if (size != 2) {
       printf(
           "This application is meant to be run with 2 MPI processes, not %d.\n",
           size);
       MPI_Abort(comm, EXIT_FAILURE);
     }

     // işlemin sırasını alıyoruz
     int rank;
     MPI_Comm_rank(comm, &rank);

     // pencerede kullanacağımız belleği ayırıyoruz ve pencereyi yaratıyoruz
     int window_buffer[4] = {0};
     if (rank == 1) {
         window_buffer[0] = 42;
         window_buffer[1] = 88;
         window_buffer[2] = 12;
         window_buffer[3] = 3;
     }
     MPI_Win win;
     MPI_Win_create(&window_buffer, (MPI_Aint)4 * sizeof(int), sizeof(int),
                    MPI_INFO_NULL, comm, &win);

     // dönemi başlatıyoruz
     MPI_Win_fence(0, win);

     int getbuf[4];
     if (rank == 0) {
           // istediğimiz değerin birinci işelmin penceresinden alıyoruz
       MPI_Get(&getbuf, 4, MPI_INT, 1, 0, 4, MPI_INT, win);
     }

     // dönemi bitiriyoruz
     MPI_Win_fence(0, win);

     if (rank == 0) {
       printf("[MPI process 0] Value fetched from MPI process 1 window:");
       for (int i = 0; i < 4; ++i) {
          printf(" %d", getbuf[i]);
       }
       printf("\n");
     }

     // pencere objesini yok ediyoruz
     MPI_Win_free(&win);

     MPI_Finalize();

     return 0;
   }

Pasif Senkronizasyon Örnek
^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: c

   #include "mpi.h" 
   #include "stdio.h"
   #include "stdlib.h"

   /* 2 işlemle pasif RMA örneği */

   #define SIZE1 100
   #define SIZE2 200

   int main(int argc, char *argv[]) 
   { 
       int rank, nprocs, A[SIZE2], B[SIZE2], i;
       MPI_Win win;
       int errs = 0;

       MPI_Init(&argc,&argv); 
       MPI_Comm_size(MPI_COMM_WORLD,&nprocs); 
       MPI_Comm_rank(MPI_COMM_WORLD,&rank); 

       if (nprocs != 2) {
           printf("Run this program with 2 processes\n");fflush(stdout);
           MPI_Abort(MPI_COMM_WORLD,1);
       }

       if (rank == 0) {
           for (i=0; i<SIZE2; i++) A[i] = B[i] = i;
           MPI_Win_create(NULL, 0, 1, MPI_INFO_NULL, MPI_COMM_WORLD, &win); 

           for (i=0; i<SIZE1; i++) {
               MPI_Win_lock(MPI_LOCK_SHARED, 1, 0, win);
               MPI_Put(A+i, 1, MPI_INT, 1, i, 1, MPI_INT, win);
               MPI_Win_unlock(1, win);
           }

           for (i=0; i<SIZE1; i++) {
               MPI_Win_lock(MPI_LOCK_SHARED, 1, 0, win);
               MPI_Get(B+i, 1, MPI_INT, 1, SIZE1+i, 1, MPI_INT, win);
               MPI_Win_unlock(1, win);
           }

           MPI_Win_free(&win);

           for (i=0; i<SIZE1; i++) 
               if (B[i] != (-4)*(i+SIZE1)) {
                   printf("Get Error: B[%d] is %d, should be %d\n", i, B[i], (-4)*(i+SIZE1));fflush(stdout);
                   errs++;
               }
       }
       else {  /* rank=1 */
           for (i=0; i<SIZE2; i++) B[i] = (-4)*i;
           MPI_Win_create(B, SIZE2*sizeof(int), sizeof(int), MPI_INFO_NULL, MPI_COMM_WORLD, &win);

           MPI_Win_free(&win); 

           for (i=0; i<SIZE1; i++) {
               if (B[i] != i) {
                   printf("Put Error: B[%d] is %d, should be %d\n", i, B[i], i);
                   errs++;
               }
           }
       }

       MPI_Finalize(); 
       return errs; 
   }
