
import javax.swing.*;
import java.awt.BorderLayout;
import java.awt.FlowLayout;
import java.awt.event.*;
import java.util.*;

// Interface
interface Monitorable { void startMonitoring(); void stopMonitoring(); }

// Abstract Patient
abstract class Patient implements Monitorable {
    final String id, name; final int age;
    final java.util.List<String> history = Collections.synchronizedList(new ArrayList<>());
    Thread monitor;
    Patient(String id,String name,int age){ this.id=id; this.name=name; this.age=age; }
    abstract double calculateBill();
    public void stopMonitoring(){ if(monitor!=null) monitor.interrupt(); }
    public String toString(){ return id+" - "+name; }
    public void startMonitorThread(Runnable r){
        monitor = new Thread(r, "Monitor-" + id + "-" + System.currentTimeMillis());
        monitor.setDaemon(true);
        monitor.start();
    }
}

// InPatient
class InPatient extends Patient {
    final int days; final double rate;
    InPatient(String id,String n,int a,int d,double r){ super(id,n,a); days=d; rate=r; }
    double calculateBill(){ return days*rate; }
    public void startMonitoring(){
        if (monitor==null || !monitor.isAlive()){
            startMonitorThread(new VitalTask(this));
        }
    }
}

// OutPatient
class OutPatient extends Patient {
    final double fee;
    OutPatient(String id,String n,int a,double f){ super(id,n,a); fee=f; }
    double calculateBill(){ return fee; }
    public void startMonitoring(){
        if (monitor==null || !monitor.isAlive()){
            startMonitorThread(new VitalTask(this));
        }
    }
}

// Custom exception
class VitalException extends Exception { VitalException(String m){ super(m); } }

// Runnable that simulates vitals (clean, no debug)
class VitalTask implements Runnable {
    private final Patient p; private final Random r = new Random();
    VitalTask(Patient p){ this.p = p; }
    public void run(){
        try{
            while(!Thread.currentThread().isInterrupted()){
                int hr = 50 + r.nextInt(81); int spo2 = 85 + r.nextInt(16);
                double t = Math.round((35.5 + r.nextDouble()*3.0)*10)/10.0;
                if(hr<40||hr>130) throw new VitalException("HR abnormal: "+hr);
                if(spo2<88) throw new VitalException("Low SpO2: "+spo2);
                String msg = String.format("%s | HR:%d SpO2:%d T:%.1f", p.name, hr, spo2, t);
                p.history.add(msg);
                CompactHealthcare.appendOutput(msg);
                Thread.sleep(2000 + r.nextInt(2001));
            }
        } catch(VitalException vx){
            String a = "ALERT for " + p.name + ": " + vx.getMessage();
            p.history.add(a); CompactHealthcare.appendOutput(a);
        } catch(InterruptedException ignored){
            // thread stopped
        }
    }
}

// GUI + app
public class CompactHealthcare extends JFrame implements ActionListener {
    static JTextArea out = new JTextArea(14,48);
    DefaultListModel<Patient> model = new DefaultListModel<>();
    JList<Patient> list = new JList<>(model);
    JTextField idF=new JTextField(4), nameF=new JTextField(8), ageF=new JTextField(3),
              daysF=new JTextField(3), rateF=new JTextField(4), feeF=new JTextField(4);

    CompactHealthcare(){
        super("Compact Healthcare");
        setDefaultCloseOperation(EXIT_ON_CLOSE); setLayout(new BorderLayout(6,6));

        JPanel top = new JPanel(new FlowLayout(FlowLayout.LEFT));
        top.add(new JLabel("ID")); top.add(idF);
        top.add(new JLabel("Name")); top.add(nameF);
        top.add(new JLabel("Age")); top.add(ageF);
        top.add(new JLabel("Days")); top.add(daysF);
        top.add(new JLabel("Rate")); top.add(rateF);
        top.add(new JLabel("Fee")); top.add(feeF);

        JButton addIn=new JButton("Add In"), addOut=new JButton("Add Out"),
                start=new JButton("Start"), stop=new JButton("Stop"), hist=new JButton("History");
        addIn.addActionListener(e->addIn()); addOut.addActionListener(e->addOut());
        start.addActionListener(e->startMonitor()); stop.addActionListener(e->stopMonitor());
        hist.addActionListener(e->showHistory());
        top.add(addIn); top.add(addOut); top.add(start); top.add(stop); top.add(hist);
        add(top, BorderLayout.NORTH);

        list.setSelectionMode(ListSelectionModel.MULTIPLE_INTERVAL_SELECTION);
        add(new JScrollPane(list), BorderLayout.WEST);
        out.setEditable(false); add(new JScrollPane(out), BorderLayout.CENTER);

        pack(); setLocationRelativeTo(null); setVisible(true);
    }

    void addIn(){
        try{
            Patient p = new InPatient(idF.getText().trim(), nameF.getText().trim(),
                    Integer.parseInt(ageF.getText()), Integer.parseInt(daysF.getText()),
                    Double.parseDouble(rateF.getText()));
            model.addElement(p); appendOutput("Added InPatient "+p);
        }catch(Exception e){ appendOutput("Enter valid values for InPatient"); }
    }
    void addOut(){
        try{
            Patient p = new OutPatient(idF.getText().trim(), nameF.getText().trim(),
                    Integer.parseInt(ageF.getText()), Double.parseDouble(feeF.getText()));
            model.addElement(p); appendOutput("Added OutPatient "+p);
        }catch(Exception e){ appendOutput("Enter valid values for OutPatient"); }
    }

    void startMonitor(){
        List<Patient> selected = list.getSelectedValuesList();
        if(selected.isEmpty()){
            if(model.isEmpty()){ appendOutput("No patients to start"); return; }
            for(int i=0;i<model.size();i++){
                Patient p = model.getElementAt(i);
                p.startMonitoring();
                appendOutput("Started "+p+" | Bill: "+p.calculateBill());
            }
        } else {
            for(Patient p:selected){
                p.startMonitoring();
                appendOutput("Started "+p+" | Bill: "+p.calculateBill());
            }
        }
    }

    void stopMonitor(){
        List<Patient> selected = list.getSelectedValuesList();
        if(selected.isEmpty()){
            if(model.isEmpty()){ appendOutput("No patients to stop"); return; }
            for(int i=0;i<model.size();i++){
                Patient p = model.getElementAt(i);
                p.stopMonitoring();
                appendOutput("Stopped "+p);
            }
        } else {
            for(Patient p:selected){
                p.stopMonitoring();
                appendOutput("Stopped "+p);
            }
        }
    }

    void showHistory(){
        Patient p = list.getSelectedValue(); if(p==null){ appendOutput("Select patient"); return; }
        appendOutput("---History for "+p.name+"---");
        synchronized(p.history){ for(String s: p.history) appendOutput(s); }
    }

    public void actionPerformed(ActionEvent e){}

    static void appendOutput(String s){ SwingUtilities.invokeLater(() -> out.append(s + "\n")); }

    public static void main(String[] args){ SwingUtilities.invokeLater(CompactHealthcare::new); }
}

