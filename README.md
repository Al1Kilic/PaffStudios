Kişilik Testi Uygulaması
Bu proje, kullanıcıların kişilik analizleri yapabilmesi için yapay zeka tarafından oluşturulmuş kişilik testini içermektedir. Test, AI tarafından üretilen sorularla birlikte her bir soru için görseller de sunar. Testi tamamladıktan sonra, kullanıcıların cevaplarına göre kişilik analizleri yapılır.

Özellikler
Yapay Zeka Tarafından Üretilen Sorular: Test, yapay zeka modelini (Llama 3) kullanarak 10 sorudan oluşur ve her soru 4 farklı seçenekle sunulur.
Unsplash'tan Görsel Çekme: Her soru, ilgili konuyu görselleştiren bir görselle birlikte gelir. Görseller, Unsplash API kullanılarak dinamik olarak çekilir.
Kişilik Analizi: Test sonunda, kullanıcının verdiği yanıtlara dayalı olarak kişilik analizi yapılır.
Türkçe Çeviri: Quiz ve kişilik analizi tamamen Türkçe'ye çevrilmiştir.
Gereksinimler
Projeyi çalıştırabilmek için aşağıdaki bağımlılıkları yüklemeniz gerekmektedir:
pip install nltk requests torch tkinter langchain-ollama deep-translator diffusers Pillow ttkbootstrap
