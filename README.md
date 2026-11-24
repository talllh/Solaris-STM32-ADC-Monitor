Solaris STM32 ADC Monitor & KiCad SchematicDokuz Eylül Üniversitesi Solaris Güneş Enerjili Araç Takımı — Elektronik Alt Grubu kapsamında geliştirilen bu proje; STM32 Nucleo-F103RB mikrodenetleyicisi üzerinden analog veri okuma (ADC), gerçek zamanlı akım/direnç hesaplaması, eşik tabanlı LED kontrolü ve devrenin KiCad şematik tasarımını içermektedir.1. Projenin AmacıGüneş enerjili araç alt sistemlerinde (gaz pedalı konum sensörü, batarya gerilim izleme ve analog telemetri girişleri) kullanılan Analog-Dijital Dönüşüm (ADC) mimarisini uygulamak, mikrodenetleyici üzerinden dolaylı akım ve eşdeğer direnç hesaplaması gerçekleştirmek ve belirlenen kritik eşik gerilimi ($2.64\text{ V}$) aşıldığında donanımsal uyarı üretmektir.2. Donanım ve Devre TopolojisiMikrodenetleyici giriş pinleri yüksek empedanslı olduğu için doğrudan ampermetre gibi seri bağlanarak akım ölçemez. Bu nedenle devreden geçen ortak akımı fiziksel olarak tespit edebilmek amacıyla reosta (değişken direnç) + sabit yük direnci seri topolojisi kurulmuştur:Mikrodenetleyici: STM32 Nucleo-F103RB (STM32F103RBT6, ARM Cortex-M3, 12-bit ADC)Potansiyometre ($RV_1 = 10\text{ k}\Omega$): 1 numaralı bacağı +3.3V hattına bağlıdır; 3 numaralı bacağı paralel kaçak yolları önlemek amacıyla boşta (No-Connect) bırakılarak 2 bacaklı değişken direnç modunda kullanılmıştır.Sabit Yük Direnci ($R_1 = 20\text{ k}\Omega$): Potansiyometrenin 2 numaralı (orta) bacağı ile GND arasına seri bağlanmıştır. $20\text{ k}\Omega$ seçimi, potansiyometre üzerindeki yükleme etkisini (loading effect) azaltarak doğrusal gerilim dağılımı sağlar.Ölçüm Düğümü (PA0 / A0): Potansiyometrenin orta bacağı ile $20\text{ k}\Omega$ direncin birleşim düğümü doğrudan ADC1_IN0 (PA0) girişine bağlanmıştır.Durum Göstergesi (PA5 / LD2): Düğüm gerilimi $2.64\text{ V}$ eşik değerini aştığında yanan dijital çıkış LED'idir.3. Depo Klasör YapısıSolaris-STM32-ADC-Monitor/
├── Firmware/                  # STM32CubeIDE proje dosyaları (Core, Drivers, .ioc)
├── Hardware/                  # KiCad proje dosyaları (.kicad_pro, .kicad_sch)
└── README.md                  # Proje teknik dokümantasyonu
4. Matematiksel Model ve Çalışma MantığıPA0 pininden 12-bit çözünürlükte ($0 - 4095$) okunan ham ADC verisi, referans gerilimi ($3.3\text{ V}$) ile ölçeklenerek $20\text{ k}\Omega$ direnç üzerindeki gerilim düşümü ($V_{20\text{k}}$) hesaplanır:
$$V_{20\text{k}} = \left(\frac{\text{ADC}_{\text{raw}}}{4095}\right) \cdot 3.3\text{ V}$$Seri devredeki ortak akım ($I$), toplam eşdeğer direnç ($R_{\text{toplam}}$) ve potansiyometrenin anlık direnci ($R_{\text{pot}}$), Ohm Yasası kullanılarak elde edilir:
$$I = \frac{V_{20\text{k}}}{20000\,\Omega} \quad , \quad R_{\text{toplam}} = \frac{3.3\text{ V}}{I} \quad , \quad R_{\text{pot}} = R_{\text{toplam}} - 20000\,\Omega$$Potansiyometre konumuna göre düğüm gerilimi $2.20\text{ V}$ ile $3.30\text{ V}$ aralığında değişir. Orta nokta eşiği olan $2.64\text{ V}$ aşıldığında LD2 LED'i aktif hale getirilir.5. Uygulanan Gömülü C Kodu (Firmware/Core/Src/main.c)/* USER CODE BEGIN PV */
uint16_t adc_ham_deger = 0;          // 0 - 4095 ham ADC verisi
float direnc_voltaji_V = 0.0f;       // 20k direnc uzerindeki gerilim (V)
float devre_akimi_mA = 0.0f;         // Seri devreden gecen ortak akim (mA)
float toplam_direnc_Ohm = 0.0f;      // R_pot + R_20k toplam direnc (Ohm)
float sadece_pot_direnci_Ohm = 0.0f; // Potansiyometrenin anlik direnci (Ohm)

const float SABIT_DIRENC_OHM = 20000.0f; // 20k Ohm sabit yuk direnci
const float BESLEME_VOLTAJI = 3.3f;      // 3.3V sistem besleme gerilimi
const float ESIK_VOLTAJ = 2.64f;         // Orta nokta esik gerilimi (V)
/* USER CODE END PV */

/* USER CODE BEGIN 3 */
HAL_ADC_Start(&hadc1);

if (HAL_ADC_PollForConversion(&hadc1, 10) == HAL_OK)
{
    adc_ham_deger = HAL_ADC_GetValue(&hadc1);

    // 1. 20k direnc uzerindeki gerilim farki (PA0 - GND)
    direnc_voltaji_V = ((float)adc_ham_deger / 4095.0f) * BESLEME_VOLTAJI;

    // 2. Seri koldan gecen ortak akim (Amper ve mA)
    float akim_Amper = direnc_voltaji_V / SABIT_DIRENC_OHM;
    devre_akimi_mA = akim_Amper * 1000.0f;

    // 3. Toplam direnc ve potansiyometre direnci hesabi (sifira bolme korumali)
    if (akim_Amper > 0.000001f)
    {
        toplam_direnc_Ohm = BESLEME_VOLTAJI / akim_Amper;
        sadece_pot_direnci_Ohm = toplam_direnc_Ohm - SABIT_DIRENC_OHM;
    }
    else
    {
        toplam_direnc_Ohm = 999999.0f;
        sadece_pot_direnci_Ohm = 999999.0f;
    }

    // 4. Threshold (Esik) gerilimi kontrolu
    if (direnc_voltaji_V > ESIK_VOLTAJ)
    {
        HAL_GPIO_WritePin(LD2_GPIO_Port, LD2_Pin, GPIO_PIN_SET);
    }
    else
    {
        HAL_GPIO_WritePin(LD2_GPIO_Port, LD2_Pin, GPIO_PIN_RESET);
    }
}

HAL_ADC_Stop(&hadc1);
HAL_Delay(50);
/* USER CODE END 3 */
6. Derleme ve ÇalıştırmaFirmware/ klasörü içindeki .ioc veya .project dosyasını STM32CubeIDE ile içe aktarın (File > Open Projects from File System).Debug modunda projeyi derleyip kart üzerine yükleyin.Live Expressions sekmesine direnc_voltaji_V, devre_akimi_mA ve sadece_pot_direnci_Ohm değişkenlerini ekleyerek potansiyometre hareketini canlı izleyin.
