package com.fanremote

import android.app.Activity
import android.content.Context
import android.content.SharedPreferences
import android.hardware.ConsumerIrManager
import android.os.Bundle
import android.os.Handler
import android.os.Looper
import android.text.InputType
import android.view.ViewGroup
import android.view.WindowManager
import android.widget.Button
import android.widget.EditText
import android.widget.LinearLayout
import android.widget.ScrollView
import android.widget.TextView

class MainActivity : Activity() {

    private var ir: ConsumerIrManager? = null
    private val handler = Handler(Looper.getMainLooper())
    private lateinit var prefs: SharedPreferences
    private lateinit var status: TextView
    private lateinit var etA1: EditText
    private lateinit var etA2: EditText
    private lateinit var etC1: EditText
    private lateinit var etC2: EditText
    private lateinit var etDelay: EditText
    private lateinit var etFreq: EditText
    private lateinit var etTimes: EditText
    private lateinit var etRaw: EditText

    private var aStart = 0
    private var aEnd = 255
    private var cStart = 0
    private var cEnd = 15
    private var delayMs = 500L
    private var freq = 38000
    private var times = 2
    private var cur = 0
    private var next = 0
    private var running = false
    private val names = listOf("ON/OFF", "1", "2", "3", "4", "5")
    private val remoteBtns = HashMap<String, Button>()

    // ---------------- scanning ----------------

    private val stepper = object : Runnable {
        override fun run() {
            if (!running) return
            if (next >= total()) {
                running = false
                prefs.edit().remove("pos").apply()
                status.text = "An gama bincike, babu amsa. Gwada wani range ko karin sau aiko."
                return
            }
            cur = next
            next = cur + 1
            prefs.edit().putInt("pos", cur).apply()
            sendIdx(cur)
            handler.postDelayed(this, delayMs)
        }
    }

    private fun dp(v: Int) = (v * resources.displayMetrics.density).toInt()
    private fun total() = (aEnd - aStart + 1) * (cEnd - cStart + 1)
    private fun addrAt(i: Int) = aStart + i / (cEnd - cStart + 1)
    private fun cmdAt(i: Int) = cStart + i % (cEnd - cStart + 1)

    // ---------------- IR ----------------

    private fun necPattern(address: Int, command: Int): IntArray {
        val p = ArrayList<Int>()
        p.add(9000); p.add(4500)
        val bytes = intArrayOf(
            address and 0xFF, address.inv() and 0xFF,
            command and 0xFF, command.inv() and 0xFF
        )
        for (b in bytes) {
            for (i in 0 until 8) {
                p.add(560)
                p.add(if (((b shr i) and 1) == 1) 1690 else 560)
            }
        }
        p.add(560)       // final burst ON
        p.add(40000)     // trailing OFF: array now starts ON, ends OFF, even count
        return p.toIntArray()
    }

    // repeat whole frame n times (frame already ends with long OFF = inter-frame gap)
    private fun repeatPattern(frame: IntArray, n: Int): IntArray {
        val out = ArrayList<Int>()
        for (k in 0 until n) for (v in frame) out.add(v)
        return out.toIntArray()
    }

    private fun transmit(pattern: IntArray) {
        if (ir?.hasIrEmitter() != true) {
            status.text = "Wannan wayar ba ta da IR blaster!"
            running = false
            return
        }
        try {
            ir?.transmit(freq, pattern)
        } catch (e: Exception) {
            status.text = "Kuskure: " + e.message
            running = false
        }
    }

    private fun send(a: Int, c: Int) {
        transmit(repeatPattern(necPattern(a, c), times))
    }

    private fun sendIdx(i: Int) {
        val a = addrAt(i)
        val c = cmdAt(i)
        send(a, c)
        status.text = "Code ${i + 1}/${total()}  |  address=$a  command=$c"
    }

    private fun sendRaw() {
        val parts = etRaw.text.toString()
            .split(",")
            .mapNotNull { it.trim().toIntOrNull() }
        if (parts.size < 3) {
            status.text = "RAW format: freq,on,off,on,off,...  (misali: 38000,9000,4500,560,560)"
            return
        }
        val rawFreq = parts[0]
        val pattern = parts.drop(1).toIntArray()
        try {
            ir?.transmit(rawFreq, pattern)
            status.text = "An aika RAW a $rawFreq Hz (${pattern.size} readings)"
        } catch (e: Exception) {
            status.text = "Kuskure: " + e.message
        }
    }

    // ---------------- fields / buttons ----------------

    private fun num(et: EditText, def: Int, lo: Int = 0, hi: Int = 255): Int =
        (et.text.toString().toIntOrNull() ?: def).coerceIn(lo, hi)

    private fun readFields() {
        val a1 = num(etA1, 0)
        val a2 = num(etA2, 255)
        val c1 = num(etC1, 0)
        val c2 = num(etC2, 15)
        aStart = minOf(a1, a2)
        aEnd = maxOf(a1, a2)
        cStart = minOf(c1, c2)
        cEnd = maxOf(c1, c2)
        delayMs = (etDelay.text.toString().toLongOrNull() ?: 500L).coerceIn(100L, 5000L)
        freq = num(etFreq, 38000, 20000, 60000)
        times = num(etTimes, 2, 1, 5)
    }

    private fun stop() {
        running = false
        handler.removeCallbacks(stepper)
    }

    private fun sendSaved(name: String) {
        val s = prefs.getString(name, null)
        if (s == null) {
            status.text = "Ba a ajiye $name ba tukuna"
            return
        }
        val parts = s.split(",")
        readFields()
        send(parts[0].toInt(), parts[1].toInt())
        status.text = "An aika $name"
    }

    private fun refreshRemote() {
        for (n in names) {
            val saved = prefs.getString(n, null) != null
            remoteBtns[n]?.text = if (saved) "$n  ✓" else n
        }
    }

    private fun addText(parent: LinearLayout, t: String, size: Float = 14f): TextView {
        val tv = TextView(this)
        tv.text = t
        tv.textSize = size
        tv.setPadding(0, dp(10), 0, dp(4))
        parent.addView(tv)
        return tv
    }

    private fun addBtn(parent: LinearLayout, t: String, onClick: () -> Unit): Button {
        val b = Button(this)
        b.text = t
        b.setOnClickListener { onClick() }
        if (parent.orientation == LinearLayout.HORIZONTAL) {
            b.layoutParams = LinearLayout.LayoutParams(0, ViewGroup.LayoutParams.WRAP_CONTENT, 1f)
        }
        parent.addView(b)
        return b
    }

    private fun addRow(parent: LinearLayout): LinearLayout {
        val r = LinearLayout(this)
        r.orientation = LinearLayout.HORIZONTAL
        parent.addView(r)
        return r
    }

    private fun addNum(row: LinearLayout, hint: String, def: String): EditText {
        val et = EditText(this)
        et.inputType = InputType.TYPE_CLASS_NUMBER
        et.hint = hint
        et.setText(def)
        et.layoutParams = LinearLayout.LayoutParams(0, ViewGroup.LayoutParams.WRAP_CONTENT, 1f)
        row.addView(et)
        return et
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        ir = getSystemService(Context.CONSUMER_IR_SERVICE) as? ConsumerIrManager
        prefs = getSharedPreferences("codes", MODE_PRIVATE)
        window.addFlags(WindowManager.LayoutParams.FLAG_KEEP_SCREEN_ON)

        val root = LinearLayout(this)
        root.orientation = LinearLayout.VERTICAL
        root.setPadding(dp(16), dp(16), dp(16), dp(24))
        val scroll = ScrollView(this)
        scroll.addView(root)
        setContentView(scroll)

        addText(root, "Fan Remote", 24f)
        status = addText(root, "")
        val has = ir?.hasIrEmitter() == true
        status.text = if (has) "IR blaster yana nan. Fara bincike." else "Wannan wayar ba ta da IR blaster!"

        addText(root, "1) Bincike (address, command, jinkiri, frequency, sau nawa)")
        val r1 = addRow(root)
        etA1 = addNum(r1, "addr daga", "0")
        etA2 = addNum(r1, "addr zuwa", "255")
        val r2 = addRow(root)
        etC1 = addNum(r2, "cmd daga", "0")
        etC2 = addNum(r2, "cmd zuwa", "15")
        val r3 = addRow(root)
        etDelay = addNum(r3, "jinkiri ms", "500")
        etFreq = addNum(r3, "frequency Hz", "38000")
        etTimes = addNum(root, "sau nawa a aika (1-5)", "2")

        val r4 = addRow(root)
        addBtn(r4, "FARA") {
            readFields()
            cur = 0
            next = 0
            prefs.edit().putInt("pos", 0).apply()
            running = true
            handler.post(stepper)
        }
        addBtn(r4, "TSAYA") {
            stop()
            prefs.edit().putInt("pos", cur).apply()
            status.text = "An tsaya a code ${cur + 1}: address=${addrAt(cur)} command=${cmdAt(cur)}"
        }
        addBtn(r4, "CI GABA") {
            readFields()
            if (!running) {
                cur = prefs.getInt("pos", 0)
                next = cur
                running = true
                handler.post(stepper)
            }
        }

        addText(root, "Idan fan ya amsa, danna TSAYA, sai ka yi amfani da BAYA/GABA har sai ka sami ainihin code ɗin da ya sa fan ya amsa.")
        val r5 = addRow(root)
        addBtn(r5, "◀ BAYA") {
            stop()
            cur = maxOf(cur - 1, 0)
            sendIdx(cur)
        }
        addBtn(r5, "SAKE AIKAWA") {
            stop()
            sendIdx(cur)
        }
        addBtn(r5, "GABA ▶") {
            stop()
            cur = minOf(cur + 1, total() - 1)
            sendIdx(cur)
        }

        addText(root, "2) Ajiye code ɗin da ke kan allo a matsayin:")
        var row = addRow(root)
        for ((i, n) in names.withIndex()) {
            if (i == 3) row = addRow(root)
            addBtn(row, n) {
                prefs.edit().putString(n, "${addrAt(cur)},${cmdAt(cur)}").apply()
                status.text = "An ajiye $n = address ${addrAt(cur)}, command ${cmdAt(cur)}"
                refreshRemote()
            }
        }

        addText(root, "3) Remote ɗinka")
        for (n in names) {
            val b = addBtn(root, n) { sendSaved(n) }
            remoteBtns[n] = b
        }
        refreshRemote()

        addText(root, "4) RAW pattern (idan an decode remote ɗin)")
        etRaw = EditText(this)
        etRaw.hint = "38000,9000,4500,560,560,..."
        etRaw.minLines = 2
        etRaw.setText(prefs.getString("lastraw", ""))
        root.addView(etRaw)
        addBtn(root, "AIKA RAW") {
            prefs.edit().putString("lastraw", etRaw.text.toString()).apply()
            sendRaw()
        }
    }

    override fun onDestroy() {
        stop()
        super.onDestroy()
    }
}
