//+------------------------------------------------------------------+
//|                                                      MoneyManager.mq4 |
//|                        Copyright 2023, MetaQuotes Software Corp. |
//|                                             https://www.metaquotes.net/ |
//+------------------------------------------------------------------+
#property copyright "Copyright 2023, MetaQuotes Software Corp."
#property link      "https://www.metaquotes.net/"
#property version   "1.00"
#property strict
#property script_show_inputs

//+------------------------------------------------------------------+
//| Input Variables                                                  |
//+------------------------------------------------------------------+
input string Q1 = "آیا تحلیل تکنیکال فعلی سیگنال خرید/فروش می‌دهد؟"; // بله/خیر
input string Q2 = "آیا شرایط بازار مناسب است؟"; // بله/خیر
input string Q3 = "آیا اخبار مهمی در راه است؟"; // بله/خیر
input string Q4 = "آیا حجم معاملات کافی است؟"; // بله/خیر
input string Q5 = "آیا استراتژی شما این معامله را تایید می‌کند؟"; // بله/خیر

input double RiskPercent = 0.5; // درصد ریسک از حساب
input double FixedDollarRisk = 100; // ریسک ثابت به دلار
input bool AutoCalculate = true; // محاسبه خودکار حجم معامله
input int Slippage = 3; // حداکثر لغزش مجاز
input int MagicNumber = 12345; // شماره جادویی برای شناسایی معاملات

//+------------------------------------------------------------------+
//| Global Variables                                                 |
//+------------------------------------------------------------------+
double EntryPrice = 0;
double StopLossPrice = 0;
double TakeProfitPrice = 0;
bool PendingBuySelected = false;
bool PendingSellSelected = false;
color BuyColor = clrGreen;
color SellColor = clrRed;
color PendingBuyColor = clrLightGreen;
color PendingSellColor = clrLightPink;

//+------------------------------------------------------------------+
//| Expert initialization function                                   |
//+------------------------------------------------------------------+
int OnInit()
{
   // بررسی شرایط اولیه
   if(!CheckConditions())
   {
      Alert("شرایط معامله فراهم نیست!");
      return(INIT_FAILED);
   }
   
   // ایجاد رابط کاربری
   CreateMainPanel();
   CreateRiskPanel();
   DisplayAccountInfo();
   
   return(INIT_SUCCEEDED);
}

//+------------------------------------------------------------------+
//| Expert deinitialization function                                 |
//+------------------------------------------------------------------+
void OnDeinit(const int reason)
{
   // حذف تمام آبجکت‌های گرافیکی
   ObjectsDeleteAll(0);
}

//+------------------------------------------------------------------+
//| Expert tick function                                             |
//+------------------------------------------------------------------+
void OnTick()
{
   // بروزرسانی اطلاعات حساب
   DisplayAccountInfo();
   
   // اگر خطوط تغییر کرده‌اند، حجم را مجددا محاسبه کن
   static double lastSL = 0;
   if(lastSL != StopLossPrice && AutoCalculate)
   {
      lastSL = StopLossPrice;
      double lotSize = CalculateLotSize();
      Comment("حجم معامله محاسبه شده: ", lotSize);
   }
}

//+------------------------------------------------------------------+
//| Chart Event function                                             |
//+------------------------------------------------------------------+
void OnChartEvent(const int id, const long &lparam, const double &dparam, const string &sparam)
{
   // مدیریت کلیک روی دکمه‌ها
   if(id == CHARTEVENT_OBJECT_CLICK)
   {
      if(sparam == "BUY_BTN") PlaceMarketOrder(OP_BUY);
      if(sparam == "SELL_BTN") PlaceMarketOrder(OP_SELL);
      if(sparam == "PENDING_BUY_BTN") { PendingBuySelected = true; PendingSellSelected = false; }
      if(sparam == "PENDING_SELL_BTN") { PendingSellSelected = true; PendingBuySelected = false; }
      if(sparam == "SET_BTN") HandlePendingOrders();
      if(sparam == "CANCEL_BTN") { PendingBuySelected = false; PendingSellSelected = false; }
      if(sparam == "CLOSE_ALL_BTN") CloseAllTrades();
      if(sparam == "CLOSE_BUY_BTN") CloseAllTrades(OP_BUY);
      if(sparam == "CLOSE_SELL_BTN") CloseAllTrades(OP_SELL);
   }
   
   // مدیریت حرکت خطوط
   if(id == CHARTEVENT_OBJECT_DRAG)
   {
      if(sparam == "SL_LINE") StopLossPrice = ObjectGet("SL_LINE", OBJPROP_PRICE1);
      if(sparam == "TP_LINE") TakeProfitPrice = ObjectGet("TP_LINE", OBJPROP_PRICE1);
   }
}

//+------------------------------------------------------------------+
//| Custom Functions                                                 |
//+------------------------------------------------------------------+

bool CheckConditions()
{
   string reasons = "";
   
   if(Q1 != "بله") reasons += "سوال 1: تحلیل تکنیکال تایید نمی‌کند\n";
   if(Q2 != "بله") reasons += "سوال 2: شرایط بازار مناسب نیست\n";
   if(Q3 != "خیر") reasons += "سوال 3: اخبار مهم در راه است\n";
   if(Q4 != "بله") reasons += "سوال 4: حجم معاملات کافی نیست\n";
   if(Q5 != "بله") reasons += "سوال 5: استراتژی تایید نمی‌کند\n";
   
   if(reasons != "")
   {
      Alert("شرایط معامله فراهم نیست به دلایل زیر:\n", reasons);
      return false;
   }
   return true;
}

void CreateMainPanel()
{
   // ایجاد دکمه‌های اصلی
   CreateButton("BUY_BTN", 10, 50, 80, 30, "خرید", BuyColor);
   CreateButton("SELL_BTN", 100, 50, 80, 30, "فروش", SellColor);
   
   // ایجاد دکمه‌های پندینگ اوردرها
   CreateButton("PENDING_BUY_BTN", 10, 90, 80, 30, "خرید معلق", PendingBuyColor);
   CreateButton("PENDING_SELL_BTN", 100, 90, 80, 30, "فروش معلق", PendingSellColor);
   
   // دکمه‌های تنظیم و انصراف
   CreateButton("SET_BTN", 10, 130, 80, 30, "تنظیم", clrGray);
   CreateButton("CANCEL_BTN", 100, 130, 80, 30, "انصراف", clrGray);
   
   // دکمه‌های بستن معاملات
   CreateButton("CLOSE_BUY_BTN", 10, 170, 80, 30, "بستن خریدها", clrDarkGray);
   CreateButton("CLOSE_SELL_BTN", 100, 170, 80, 30, "بستن فروشها", clrDarkGray);
   CreateButton("CLOSE_ALL_BTN", 10, 210, 170, 30, "بستن همه معاملات", clrDarkSlateGray);
}

void CreateRiskPanel()
{
   // ایجاد دکمه‌های انتخاب نوع ریسک
   CreateButton("RiskType_Percent", 10, 10, 80, 20, "% سرمایه", RiskPercent > 0 ? clrBlue : clrGray);
   CreateButton("RiskType_Fixed", 100, 10, 80, 20, "دلار ثابت", FixedDollarRisk > 0 && !AutoCalculate ? clrBlue : clrGray);
   CreateButton("RiskType_Auto", 190, 10, 80, 20, "خودکار", AutoCalculate ? clrBlue : clrGray);
   
   // فیلد ورودی مقدار ریسک
   ObjectCreate(0, "RiskValue", OBJ_EDIT, 0, 0, 0);
   ObjectSetInteger(0, "RiskValue", OBJPROP_XDISTANCE, 280);
   ObjectSetInteger(0, "RiskValue", OBJPROP_YDISTANCE, 10);
   ObjectSetInteger(0, "RiskValue", OBJPROP_XSIZE, 50);
   ObjectSetInteger(0, "RiskValue", OBJPROP_YSIZE, 20);
   ObjectSetString(0, "RiskValue", OBJPROP_TEXT, DoubleToString(RiskPercent, 2));
   ObjectSetInteger(0, "RiskValue", OBJPROP_ALIGN, ALIGN_RIGHT);
}

void DisplayAccountInfo()
{
   double marginLevel = 0;
   if(AccountMargin() != 0)
   {
      marginLevel = AccountEquity() / AccountMargin() * 100;
   }
   
   ObjectCreate(0, "AccInfo", OBJ_LABEL, 0, 0, 0);
   ObjectSetInteger(0, "AccInfo", OBJPROP_XDISTANCE, 10);
   ObjectSetInteger(0, "AccInfo", OBJPROP_YDISTANCE, 250);
   ObjectSetString(0, "AccInfo", OBJPROP_TEXT, 
                  "موجودی: " + DoubleToString(AccountBalance(), 2) + 
                  " | سرمایه: " + DoubleToString(AccountEquity(), 2) +
                  " | مارجین آزاد: " + DoubleToString(AccountFreeMargin(), 2) +
                  " | سطح مارجین: " + DoubleToString(marginLevel, 2) + "%");
}

void CreateButton(string name, int x, int y, int width, int height, string text, color bgColor)
{
   ObjectCreate(0, name, OBJ_BUTTON, 0, 0, 0);
   ObjectSetInteger(0, name, OBJPROP_XDISTANCE, x);
   ObjectSetInteger(0, name, OBJPROP_YDISTANCE, y);
   ObjectSetInteger(0, name, OBJPROP_XSIZE, width);
   ObjectSetInteger(0, name, OBJPROP_YSIZE, height);
   ObjectSetString(0, name, OBJPROP_TEXT, text);
   ObjectSetInteger(0, name, OBJPROP_BGCOLOR, bgColor);
   ObjectSetInteger(0, name, OBJPROP_CORNER, CORNER_LEFT_UPPER);
   ObjectSetInteger(0, name, OBJPROP_FONTSIZE, 8);
}

double CalculateLotSize()
{
   double lotSize = 0;
   double slDistance = MathAbs(EntryPrice - StopLossPrice) / Point;
   
   if(slDistance == 0) return 0;
   
   if(AutoCalculate)
   {
      double tickValue = MarketInfo(Symbol(), MODE_TICKVALUE);
      if(tickValue == 0) return 0;
      
      lotSize = (AccountBalance() * RiskPercent / 100) / (slDistance * tickValue);
   }
   else if(FixedDollarRisk > 0)
   {
      double tickValue = MarketInfo(Symbol(), MODE_TICKVALUE);
      if(tickValue == 0) return 0;
      
      lotSize = FixedDollarRisk / (slDistance * tickValue);
   }
   
   // محدود کردن حجم به مقادیر مجاز
   double minLot = MarketInfo(Symbol(), MODE_MINLOT);
   double maxLot = MarketInfo(Symbol(), MODE_MAXLOT);
   double lotStep = MarketInfo(Symbol(), MODE_LOTSTEP);
   
   lotSize = MathMax(minLot, MathMin(maxLot, lotSize));
   lotSize = NormalizeDouble(lotSize / lotStep, 0) * lotStep;
   
   return lotSize;
}

void PlaceMarketOrder(int cmd)
{
   double lotSize = CalculateLotSize();
   if(lotSize <= 0) 
   {
      Alert("حجم معامله نامعتبر است!");
      return;
   }
   
   EntryPrice = (cmd == OP_BUY) ? Ask : Bid;
   StopLossPrice = (cmd == OP_BUY) ? EntryPrice - 100 * Point : EntryPrice + 100 * Point;
   TakeProfitPrice = (cmd == OP_BUY) ? EntryPrice + 200 * Point : EntryPrice - 200 * Point;
   
   int ticket = OrderSend(Symbol(), cmd, lotSize, EntryPrice, Slippage, StopLossPrice, TakeProfitPrice, "", MagicNumber);
   if(ticket > 0)
   {
      DrawTradeLines(EntryPrice, StopLossPrice, TakeProfitPrice);
      Print("معامله با موفقیت انجام شد. شماره: ", ticket);
   }
   else
   {
      Print("خطا در انجام معامله. کد خطا: ", GetLastError());
   }
}

void HandlePendingOrders()
{
   double lotSize = CalculateLotSize();
   if(lotSize <= 0) 
   {
      Alert("حجم معامله نامعتبر است!");
      return;
   }
   
   if(PendingBuySelected)
   {
      EntryPrice = ObjectGet("ENTRY_LINE", OBJPROP_PRICE1);
      StopLossPrice = ObjectGet("SL_LINE", OBJPROP_PRICE1);
      TakeProfitPrice = ObjectGet("TP_LINE", OBJPROP_PRICE1);
      
      int cmd = (Ask < EntryPrice) ? OP_BUYSTOP : OP_BUYLIMIT;
      int ticket = OrderSend(Symbol(), cmd, lotSize, EntryPrice, Slippage, StopLossPrice, TakeProfitPrice, "", MagicNumber);
      
      if(ticket > 0) Print("سفارش خرید معلق با موفقیت ثبت شد. شماره: ", ticket);
      else Print("خطا در ثبت سفارش خرید معلق. کد خطا: ", GetLastError());
   }
   
   if(PendingSellSelected)
   {
      EntryPrice = ObjectGet("ENTRY_LINE", OBJPROP_PRICE1);
      StopLossPrice = ObjectGet("SL_LINE", OBJPROP_PRICE1);
      TakeProfitPrice = ObjectGet("TP_LINE", OBJPROP_PRICE1);
      
      int cmd = (Bid > EntryPrice) ? OP_SELLSTOP : OP_SELLLIMIT;
      int ticket = OrderSend(Symbol(), cmd, lotSize, EntryPrice, Slippage, StopLossPrice, TakeProfitPrice, "", MagicNumber);
      
      if(ticket > 0) Print("سفارش فروش معلق با موفقیت ثبت شد. شماره: ", ticket);
      else Print("خطا در ثبت سفارش فروش معلق. کد خطا: ", GetLastError());
   }
}

void DrawTradeLines(double entry, double sl, double tp)
{
   ObjectCreate(0, "ENTRY_LINE", OBJ_HLINE, 0, 0, entry);
   ObjectSetInteger(0, "ENTRY_LINE", OBJPROP_COLOR, clrBlue);
   ObjectSetInteger(0, "ENTRY_LINE", OBJPROP_STYLE, STYLE_SOLID);
   ObjectSetInteger(0, "ENTRY_LINE", OBJPROP_WIDTH, 2);
   ObjectSetInteger(0, "ENTRY_LINE", OBJPROP_SELECTABLE, 1);
   
   ObjectCreate(0, "SL_LINE", OBJ_HLINE, 0, 0, sl);
   ObjectSetInteger(0, "SL_LINE", OBJPROP_COLOR, clrRed);
   ObjectSetInteger(0, "SL_LINE", OBJPROP_STYLE, STYLE_DOT);
   ObjectSetInteger(0, "SL_LINE", OBJPROP_WIDTH, 1);
   ObjectSetInteger(0, "SL_LINE", OBJPROP_SELECTABLE, 1);
   
   ObjectCreate(0, "TP_LINE", OBJ_HLINE, 0, 0, tp);
   ObjectSetInteger(0, "TP_LINE", OBJPROP_COLOR, clrGreen);
   ObjectSetInteger(0, "TP_LINE", OBJPROP_STYLE, STYLE_DOT);
   ObjectSetInteger(0, "TP_LINE", OBJPROP_WIDTH, 1);
   ObjectSetInteger(0, "TP_LINE", OBJPROP_SELECTABLE, 1);
}

void CloseAllTrades(int type = -1)
{
   for(int i = OrdersTotal() - 1; i >= 0; i--)
   {
      if(OrderSelect(i, SELECT_BY_POS))
      {
         if(OrderMagicNumber() != MagicNumber) continue;
         
         bool result = false;
         if((type == -1 || type == OP_BUY) && OrderType() == OP_BUY)
            result = OrderClose(OrderTicket(), OrderLots(), Bid, Slippage);
         else if((type == -1 || type == OP_SELL) && OrderType() == OP_SELL)
            result = OrderClose(OrderTicket(), OrderLots(), Ask, Slippage);
         else if((type == -1 || type > OP_SELL) && OrderType() > OP_SELL)
            result = OrderDelete(OrderTicket());
            
         if(!result)
            Print("خطا در بستن/حذف سفارش ", OrderTicket(), ". کد خطا: ", GetLastError());
      }
   }
}
