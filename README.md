/Import External class 
#include <Trade\PositionInfo.mqh>
 #include <Trade\Trade.mqh>
 #include <Trade\SymbolInfo.mqh>  
 #include <Trade\AccountInfo.mqh>
 #include <Trade\OrderInfo.mqh>

//--- معرفی متغیرهای از پیش تعریف شده برای خوانایی کد 
#define Ask     SymbolInfoDouble ( _Symbol , SYMBOL_ASK )
 #define Bid     SymbolInfoDouble ( _Symbol , SYMBOL_BID )

//--- پارامترهای ورودی 
رشته ورودی    EASettings = "-------------------------------------- -------" ؛ //-------- <تنظیمات EA> -------- ورودی ورودی       InpMagicNumber = 123456 ;    // رشته ورودی شماره جادویی    InpBotName = "LazyBot_V1" ; // رشته ورودی نام ربات   TradingSettings = "---------------------------------------- ----" ؛ //-------- <تنظیمات تجارت> -------- ورودی دو برابر    Inpuser_lot = 0.01 ;         // ورودی لات دو برابر    Inpuser_SL = 5.0 ;           //Stoploss (در پیپ) ورودی دو برابر    InpAddPrice_pip = 0 ;        //فاصله از [H]، [L] تا OP_Price (بر حسب پیپ) ورودی int       Inpuser_SLippage = 3 ;       // حداکثر لغزش allow_Pips. ورودی دو برابر    InpMax_spread = 0 ;       //حداکثر پخش مجاز (بر حسب پیپ) (0 = شناور) رشته ورودی   TimeSettings = "------------------------------- --------------" ; //-------- <Trading Time Settings> -------- input bool      isTradingTime = true ;       //اجازه دادن ورودی زمان معامله int       InpStartHour = 7 ;           // ورودی Start Hour int       InpEndHour = 22 ;            // رشته ورودی پایان ساعت   MoneyManagementSettings = "---------------------------------------- ----" ؛ //-------- <تنظیمات پول> -------- ورودی bool      isVolume_Percent = false ;    //Allow Volume Percent input double    InpRisk = 1 ;                // درصد ریسک موجودی (%)
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 

2. مقداردهی اولیه متغیرهای محلی

//پارامترهای محلی 
datetime last;
 int totalBars;
 int      Pips2Points;             // slippage 3 pips 3=points 30=point 
double   Pips2Double;     // Stoplos 15 pips 0.015 0.0150 
double slippage;
 دو برابر acSpread;
 string strComment = "" ;

CPositionInfo m_position; // شی موقعیت تجاری
CTrade m_trade; // شیء تجاری
CSymbolInfo m_symbol; // شی اطلاعات نماد
CAccountInfo m_account; // پوشش اطلاعات حساب
COrderInfo m_order; // دستورات در انتظار شیء
3. کد اصلی

با توضیح استراتژی، Everyday Bot همه سفارش‌های قدیمی را حذف می‌کند و بالاترین و کمترین مقدار نوار روزانه قبلی را پیدا می‌کند و به دو سفارش در انتظار « BUY_STOP »، « SELL_STOP » ارسال می‌کند. (بدون TakeProfit).


الف/ تابع مقداردهی اولیه

int  OnInit ()
  {
//--- 
   //تشخیص 3 یا 5 رقم 
   //پیپ و نقطه 
   اگر ( _رقم % 2 == 1 )
     {
      Pips2Double = _Point * 10 ;
      Pips2Points = 10 ;
      لغزش = 10 * Inpuser_SLippage;
     }
   دیگر
     {
      Pips2Double = _Point ;
      Pips2Points =   1 ;
      لغزش = Inpuser_SLippage;
     }
     
     if (!m_symbol.Name( Symbol ())) // بازگشت نام نماد را تنظیم می کند
       ( INIT_FAILED );
   RefreshRates();
//---
   m_trade.SetExpertMagicNumber(InpMagicNumber);
   m_trade.SetMarginMode();
   m_trade.SetTypeFillingBySymbol(m_symbol.Name());
   m_trade.SetDeviationInPoints(slippage);
//--- 
   بازگشت ( INIT_SUCCEEDED )؛
  }
ب/ عملکرد تیک خبره

void  OnTick ()
  {
     if ( TerminalInfoInteger ( TERMINAL_TRADE_ALLOWED ) == false )
     {
      نظر ( "LazyBot\nتجارت مجاز نیست." );
       برگشت ؛
     }
     
   //دریافت زمان معاملات، 
   // بخش افتتاحیه 
   // لندن 14 ساعت - 23 ساعت به وقت گرینویچ ویتنام 
   // نیویورک 19 ساعت - 04 ساعت به وقت گرینویچ ویتنام 
   MqlDateTime timeLocal;
    MqlDateTime timeServer.
   
   TimeLocal (timeLocal)؛
    TimeCurrent (timeServer)؛   
   
   // در روزهای تعطیل کار نکنید. 
   if (timeServer.day_of_week == 0 || timeServer.day_of_week == 6 )
       return ;
      
   int hourLocal = timeLocal.hour; //TimeHour(TimeLocal()); 
   int hourCurrent = timeServer.hour; //TimeHour(TimeCurrent());

   acSpread = SymbolInfoInteger ( _Symbol , SYMBOL_SPREAD );
   
   strComment = "\nساعت محلی = " + hourLocal;
   strComment += "\nساعت فعلی = " + hourCurrent;
   strComment += "\nSpread is = " + ( string )acSpread;
   strComment += "\nTotal Bars = " + ( string )totalBars;

   نظر (strComment);

   //بررسی دنباله
   TrailingSL();

//--- 
   if (آخرین != iTime (m_symbol.Name()، PERIOD_D1 ، 0 )) // && hourCurrent > InpStartHour)
     {
      //زمان معامله را بررسی کنید 
      اگر (isTradingTime)
        {
         if (hourCurrent >= InpStartHour) // && hourCurrent < InpEndHour){
           {
            DeleteOldOrds();
            //ارسال سفارش BUY_STOP و SELL_STOP
            OpenOrder();

            last = iTime (m_symbol.Name(), PERIOD_D1 , 0 );
           }
        }
      دیگر
        {
         DeleteOldOrds();
         //ارسال سفارش BUY_STOP و SELL_STOP
         OpenOrder();
         last = iTime (m_symbol.Name(), PERIOD_D1 , 0 );
        }
     }
  }
  
3.1 محاسبه سیگنال و ارسال سفارشات

//+---------------------------------------------- -------------------+ 
//| محاسبه سیگنال و ارسال سفارش | 
//+---------------------------------------------- -------------------+ 
void OpenOrder()
  {
      دو برابر TP_Buy = 0 , TP_Sell = 0 ;
       دو برابر SL_Buy = 0 , SL_Sell = 0 ;
      
      //Check Maximum Spread 
      if (InpMax_spread != 0 ){
          if (acSpread > InpMax_spread){
             Print ( __FUNCTION__ , " > گسترش فعلی بیشتر از کاربر Spread است!..." );
             برگشت ؛
         }
      }
         
      double Bar1High = m_symbol.NormalizePrice( iHigh (m_symbol.Name(), PERIOD_D1 , 1 ));
       double Bar1Low = m_symbol.NormalizePrice( iLow (m_symbol.Name(), PERIOD_D1 , 1 ));
      
      //Calculate Lots 
      double lot1 = CalculateVolume();
   
      دو برابر OpenPrice = m_symbol.NormalizePrice(Bar1High + InpAddPrice_pip * Pips2Double); // + NormalizeDouble((acSpread/Pips2Points) * Pips2Double، Digits);
   
      //برای BUY_STOP -------------------------------- 
      TP_Buy = 0 ; //Bar1High + NormalizeDouble(min_sl* Pips2Double, Digits);
      SL_Buy = m_symbol.NormalizePrice(OpenPrice - Inpuser_SL * Pips2Double);
   
      totalBars = iBars (m_symbol.Name(), PERIOD_D1 );
       نظر رشته = InpBotName + ";" + m_symbol.Name() + ";" + totalBars؛
   
         if (CheckVolumeValue(lot1)
            && CheckOrderForFREEZE_LEVEL( ORDER_TYPE_BUY_STOP ، OpenPrice)
            && CheckMoneyForTrade(m_symbol.Name()،lot1، ORDER_TYPE_BUY )
            && CheckStopLoss (OpenPrice، SL_Buy))
            {
               if (!m_trade.BuyStop(lot1، OpenPrice، m_symbol.Name()، SL_Buy، TP_Buy، ORDER_TIME_GTC ، 0 ، نظر)) // از "ORDER_TIME_GTC" در زمان انقضا استفاده کنید = 0 
               چاپ ( __FUNCTION__ "خرید، " )
            }
      
   
      //برای SELL_STOP -------------------------------- 
      OpenPrice = m_symbol.NormalizePrice(Bar1Low - InpAddPrice_pip * Pips2Double); // - NormalizeDouble((acSpread/Pips2Points) * Pips2Double, Digits);
   
      TP_Sell = 0 ; //Bar1Low - NormalizeDouble(min_sl* Pips2Double, Digits);
      SL_Sell = m_symbol.NormalizePrice(OpenPrice + Inpuser_SL * Pips2Double);
   
         if (CheckVolumeValue(lot1)
            && CheckOrderForFREEZE_LEVEL( ORDER_TYPE_SELL_STOP ، OpenPrice)
            && CheckMoneyForTrade(m_symbol.Name()،lot1، ORDER_TYPE_SELL )
            && CheckStopLoss (OpenPrice، SL_Sell))
         {
            if (!m_trade.SellStop(lot1, OpenPrice, m_symbol.Name(), SL_Sell, TP_Sell, ORDER_TIME_GTC , 0 , comment))
             چاپ ( __FUNCTION__ , "--> Sell Error" );
         }  

  }

3.2 روز جدید، تمام سفارشات قدیمی را حذف کنید

//+---------------------------------------------- -------------------+ 
//| سفارشات قدیمی دله | 
//+---------------------------------------------- -------------------+ 
void DeleteOldOrds()
  {
   string sep= ";" ;                 // جداکننده به عنوان یک کاراکتر 
   ushort u_sep;                   // کد نتیجه 
   رشته کاراکتر جداکننده [];                // آرایه ای برای گرفتن رشته ها

   for ( int i = OrdersTotal () - 1 ; i >= 0 ; i--)   // تعداد سفارشات فعلی را برمی گرداند
     {
      if (m_order.SelectByIndex(i))               // برای دسترسی بیشتر به ویژگی های آن، ترتیب در انتظار را بر اساس فهرست انتخاب می کند.
        {
         //--- کد جداکننده را دریافت کنید 
         u_sep = StringGetCharacter (Sep, 0 );
          رشته Ordcomment = m_order. اظهار نظر ()؛

         //Split OrderComment (EAName;Symbol;totalBar) برای دریافت Ordersymbol 
         int k = StringSplit (Ordcomment, u_sep, result);

         اگر (k > 2 )
           {
            string sym = m_symbol.Name();
             if ((m_order.Magic() == InpMagicNumber) && (sym == نتیجه[ 1 ]))
              {
                  m_trade.OrderDelete(m_order.Ticket());
              }
           }
        }
     }

  }
3.3 EA دارای عملکرد "Tiring StopLoss" است،  SL هر بار که قیمت تغییر می کند تغییر می کند و باعث افزایش سود می شود.

//+---------------------------------------------- -------------------+ 
//| تعقیب توقف | 
//+---------------------------------------------- -------------------+ 
void TrailingSL()
  {
   دو برابر SL_in_Pip = 0 ;

   برای ( int i = PositionsTotal () - 1 ; i >= 0 ; i--)
     {         
      if (m_position.SelectByIndex(i))      // برای دسترسی بیشتر به ویژگی‌های آن، ترتیب‌ها را بر اساس فهرست انتخاب می‌کند.
        {
         if ((m_position.Magic() == InpMagicNumber) && (m_position. Symbol () == m_symbol.Name()))    //m_symbol.Name()
           {
            double order_stoploss1 = m_position.StopLoss();
            
            // برای خرید 
            اگر (m_position.PositionType() == POSITION_TYPE_BUY )
              {                  
                  //--هنگام تغییر قیمت SL را محاسبه کنید 
                  SL_in_Pip = NormalizeDouble ((Bid - order_stoploss1), _Digits ) / Pips2Double;
                   //Print(__FUNCTION__, "Modify Order Buy" + (string)SL_in_Pip); 
                  اگر (SL_in_Pip > Inpuser_SL)
                    {
                        order_stoploss1 = NormalizeDouble (Bid - (Inpuser_SL * Pips2Double), _Digits );
                   
                        m_trade.PositionModify(m_position.Ticket()، order_stoploss1، m_position.TakeProfit());                       
                    }
              }

            //برای سفارش فروش 
            اگر (m_position.PositionType() == POSITION_TYPE_SELL )
              {
                  //--هنگام تغییر قیمت SL را محاسبه کنید 
                  SL_in_Pip = NormalizeDouble ((m_position.StopLoss() - Ask), _Digits ) / Pips2Double;
                   اگر (SL_in_Pip > Inpuser_SL)
                    {
                        order_stoploss1 = NormalizeDouble (Ask + (Inpuser_SL * Pips2Double), _Digits );
                     
                        m_trade.PositionModify(m_position.Ticket()، order_stoploss1، m_position.TakeProfit());
            
                    }
              }
           }
        }
     }
  }
3.4 مقدار حجم را بررسی کنید

//+---------------------------------------------- -------------------+ 
//| بررسی صحت حجم سفارش | 
//+---------------------------------------------- -------------------+ 
bool CheckVolumeValue ( دوبل حجم)
{
//--- حداقل حجم مجاز برای عملیات تجاری 
  double min_volume = m_symbol.LotsMin();
  
//--- حداکثر حجم مجاز عملیات تجاری 
   double max_volume = m_symbol.LotsMax();

//--- دریافت حداقل گام تغییر حجم 
   دو برابر volume_step = m_symbol.LotsStep();
   
   اگر (حجم < حداقل_حجم || حجم>حداکثر_حجم)
     {
      بازگشت ( نادرست )؛
     }

   int ratio = ( int ) MathRound (volume/volume_step);
    if ( MathAbs (نسبت*جلد_گام-حجم)> 0.0000001 )
     {
      بازگشت ( نادرست )؛
     }

   بازگشت ( درست )؛
}
3.5 FreeLevel را بررسی کنید

//+---------------------------------------------- -------------------+ 
//| بررسی سطح انجماد | 
//+---------------------------------------------- -------------------+ 
bool CheckOrderForFREEZE_LEVEL( نوع ENUM_ORDER_TYPE ، دو قیمت) //تغییر نام این تابع
{
  int freeze_level = ( int ) SymbolInfoInteger ( _Symbol ,   SYMBOL_TRADE_FREEZE_LEVEL );

   بوول چک = نادرست ;
   
//--- فقط دو نوع 
   سوئیچ سفارش را بررسی کنید (نوع)
     {
         //--- خرید 
         کیس عملیات  ORDER_TYPE_BUY_STOP :
         {  
           //--- فاصله قیمت افتتاحیه تا 
           بررسی قیمت فعال سازی را بررسی کنید = ((price-Ask) > freeze_level* _Point );
            //--- برگرداندن نتیجه بررسی 
            بازگشت (چک)؛
         }
      //--- فروش 
      کیس عملیات  ORDER_TYPE_SELL_STOP :
         {
         //--- فاصله از قیمت افتتاحیه تا 
            بررسی قیمت فعال سازی را بررسی کنید = ((Bid-price)>freeze_level* _Point );

            //--- برگرداندن نتیجه بررسی 
            بازگشت (چک)؛
         }
         زنگ تفريح ؛
     }
//--- یک تابع کمی متفاوت برای دستورات در انتظار 
   بازگشت  نادرست مورد نیاز است .
}
3.6 چک پول برای تجارت

//+---------------------------------------------- -------------------+ 
//|                                                                                  | 
//+---------------------------------------------- --------------------+

bool CheckMoneyForTrade ( رشته سیم، لات دوتایی ، نوع ENUM_ORDER_TYPE )
  {
//--- دریافت قیمت افتتاحیه 
   MqlTick mqltick;
    SymbolInfoTick (symb,mqltick)؛
    قیمت دو برابر =mqltick.ask;
    اگر (نوع== ORDER_TYPE_SELL )
      price=mqltick.bid;
//--- مقادیر حاشیه مورد نیاز و آزاد 
   دو حاشیه,free_margin= AccountInfoDouble ( ACCOUNT_MARGIN_FREE );
    //--- فراخوانی تابع بررسی 
   اگر (! OrderCalcMargin (نوع، نماد، لات، قیمت، حاشیه))
     {
      //--- مشکلی پیش آمد، گزارش و بازگشت false 
      Print ( "Error in " , __FUNCTION__ , " code=" , GetLastError ());
       بازگشت ( نادرست )؛
     }
   //--- اگر بودجه کافی برای انجام عملیات وجود ندارد 
   اگر (margin>free_margin)
     {
      //--- خطا را گزارش کنید و false را برگردانید
       Print ( "پول کافی برای " , EnumToString (نوع)، " " ,lots, " " ,symb, " Error code=" , GetLastError ());
       بازگشت ( نادرست )؛
     }
//--- بررسی 
   بازگشت موفق ( true );
}
3.7 Stoplos را بررسی کنید

//+---------------------------------------------- -------------------+ 
//|                                                                  | 
//+---------------------------------------------- --------------------+

bool CheckStopLoss ( قیمت دو برابر ، دو برابر SL)
{
//--- سطح SYMBOL_TRADE_STOPS_LEVEL را دریافت کنید 
   int stops_level = ( int ) SymbolInfoInteger (m_symbol.Name(), SYMBOL_TRADE_STOPS_LEVEL );
    if (stops_level != 0 )
     {
      PrintFormat ( "SYMBOL_TRADE_STOPS_LEVEL=%d: StopLoss و TakeProfit نباید" +
                   " نزدیک‌تر از %d امتیاز از قیمت بسته شدن باشند" , stops_level, stops_level);
     }
//--- 
   bool SL_check= false ;

   
  //--- بازگشت StopLoss را بررسی کنید SL_check = MathAbs (قیمت - SL) > (stops_level * m_symbol. Point ());  
} -
