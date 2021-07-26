
Performans Ölçme ve Eniyileme
=============================

..

   We should forget about small efficiencies, say about 97% of the time: premature optimization is the root of all evil. - Donald Knuth


Turing Ödülü sahibi meşhur Donald Knuth tarafından da belirtildiği gibi kodumuzdaki sık boğazları belirlemek yüksek performanslı programlamanın olmazsa olmazıdır. Kodumuzu optimize etmeden önce performansını ölçerek profillememiz gerekir ki optimize etmeye nerden başlayacağımızı belirleyelim. Günümüzde kodumuzun performansını ölçebileceğimiz kuvvetli araçlar bulunmaktadır. Fakat bu dokümanda Open MPI'ın sağladığı *PMPI* arayüzünü kullanarak MPI işlemlerini kendimiz profilleyeceğiz. Buna ek olarak yaptığımız hesaplamaları da profilleyerek MPI işlemlerinin bir sık boğaz oluşturup oluşturmadığını araştıracağız.

PMPI nedir?
-----------

Her standart MPI işlemini, bir ``MPI_`` veya ``PMPI_`` öneki ile çağrılabilir. Örneğin, ``MPI_Send()`` veya ``PMPI_Send()``\ 'i çağırabilirsiniz. MPI standardının bu özelliği, eşdeğer ``PMPI_`` işlevini çağıran MPI_ öneki ile işlevlerin yazılmasına izin verir. Spesifik olarak, bu şekilde yazılan bir işlev, standart işlevin davranışına ve eklemek istediğiniz diğer davranışlara sahiptir. Bu, MPI performans analizi için iki şekilde önemlidir:


#. İlk olarak, birçok performans analizi aracı *PMPI*\ 'den yararlanır. Programınız tarafından yapılan MPI aramalarını yakalarlar. PMPI işlevlerini çağırarak ilişkili mesaj iletme çağrılarını gerçekleştirirler, ancak aynı zamanda önemli performans verilerini de yakalarlar.
#. İkinci olarak, MPI davranışını özelleştirmek için bu tür sarmalayıcı işlevlerini kullanabilirsiniz. Örneğin, toplu aramalara bariyer işlemleri ekleyebilir, belirli MPI aramaları için teşhis bilgilerini yazabilirsiniz.

Profil Oluşturma
----------------

Aşağıdaki programda birden fazla işlem kullanarak $pi$ değerini hesaplıyoruz. Hesaplamanın duyarlılığını ``num_steps`` değişkeni ile belirliyoruz. Bu değişken aynı zamanda toplamda yaptığımız işlem saysını da etkiliyor.

MPI Profilleme Örnek
^^^^^^^^^^^^^^^^^^^^

.. code-block:: c

   #include <mpi.h>
   #include <stdio.h>

   static int totalBytes = 0;
   static double totalTime = 0.0;
   static double totalComputeTime = 0.0;

   static long long num_steps=100000000;
   double step;

   // okuma kolaylığı ve standarda uymak için MPI_Send fonksiyonunu tekrar tanımlıyoruz
   // bu fonksiyonun içinde ölçmek istediğimiz değerleri ölçüp global olarak tanımlanmış değişkenleri yeniliyoruz
   // Open MPI tarafından sağlanan PMPI_ ekli fonksiyonu kullanıyoruz
   int MPI_Send(const void* buffer, int count, MPI_Datatype datatype,
                int dest, int tag, MPI_Comm comm)
   {
      double tstart = MPI_Wtime();
      int size;
      int result    = PMPI_Send(buffer,count,datatype,dest,tag,comm);

      totalTime  += MPI_Wtime() - tstart;

      MPI_Type_size(datatype, &size);
      totalBytes += count*size;

      return result;
   }

   // pi hesaplaması
   double compute_pi(int num_procs, int myid){
           double x, sum = 0;
           double start = MPI_Wtime();
           step = 1.0/(double) num_steps;
           for (int i = myid; i< num_steps; i=i+num_procs){
                   x =(i+0.5)*step;
                   sum +=4.0/(1.0+x*x);
           }
           double end = MPI_Wtime();
           totalComputeTime += end - start;
           return sum;
   }

   int main(int argc, char** argv){
           int i, myid, num_procs;
           double pi=0, remote_sum, sum=0, start, end;

           MPI_Init(&argc, &argv);
           MPI_Comm_size(MPI_COMM_WORLD, &num_procs);
           MPI_Comm_rank(MPI_COMM_WORLD, &myid);

           start = MPI_Wtime();
           sum = compute_pi(num_procs, myid);

           if (myid==0){
                   for (i = 1; i< num_procs;i++){
                           MPI_Status status;
                                                   // ana işlem parçalı pi toplamlarını diğer işlemlerden alıyor
                           MPI_Recv(&remote_sum, 1, MPI_DOUBLE, i, 0, MPI_COMM_WORLD, &status);
                           sum +=remote_sum;
                   }
                   pi=sum*step;
           } else {
                                   // işlemler hesapladıkları pi değerlerini ana işleme gönderiyor
                   MPI_Send(&sum, 1, MPI_DOUBLE, 0, 0, MPI_COMM_WORLD);
           }

           MPI_Finalize();
           end = MPI_Wtime();
           if (myid ==0){
                   end = MPI_Wtime();

                   printf("Num of processors %d, total time %f, value: %lf\n", num_procs, end-start, pi);
           }
           else{
                   printf("Process [%d], total bytes %d, took %f\n", myid, totalBytes, totalTime);
                   printf("Process [%d], total compute time %f\n", myid, totalComputeTime);
           }
           return 0;
   }

Çıktı
^^^^^

.. code-block:: c

   Process [1], total bytes 8, took 0.000075
   Process [1], total comput time 0.363442
   Process [3], total bytes 8, took 0.013768
   Process [3], total comput time 0.375604
   Send-Receive: Num of processors 4, total time 0.478455, value: 3.141593
   Process [2], total bytes 8, took 0.024124
   Process [2], total comput time 0.375207

Yukarıdaki çıktıda da görüldüğü üzere programın çalışma süresinin çoğunluğunu $pi$ hesaplaması almakta. Bu durumda MPI işlemlerini optimize etmemizi gerektirecek bir durum bulunmamakta. ``num_steps`` değişkeni ile oynayarak iletişim ve hesaplama sürelerinin nasıl değiştiğini inceleyebilirsiniz.

Optimizasyon İş Akışı
---------------------

İster paralel ister seri olsun, bir kodu optimize etmek için genel iş akışı aşağıdaki gibidir:


#. Profillemek.
#. Optimize etmek.
#. Doğruluğunu kontrol etmek.
#. Verimliliği ölçmek.
#. Kazanç sağlayamayana kadar birinici adıma tekrar geri dönme.
