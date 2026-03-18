# ActivityAnalysis
C# profiling system

using NetLog.Logging;
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading;
using ModbusPolling.Events;

namespace Performance {

	/// <summary>
	/// 
	/// </summary>
	public class ActivitySummaryArgs: EventArgs {
		private string name;
		private double avg;
		private TimeSpan intv;
		private double cnt;
		public string ActivityName {
			get {
				return name;
			}
		}
		public double Average {
			get {
				return avg;
			}
		}
		public TimeSpan AverageInterval {
			get {
				return intv;
			}
		}
		public double AveragedIntervalCount {
			get {
				return cnt;
			}
		}
		public ActivitySummaryArgs( string name, double avg, TimeSpan over, double count ) {
			this.name = name;
			this.avg = avg;
			this.intv = over;
			this.cnt = count;
		}
	}

	class ActivitySummaryListenerAction: IObserver<ActivitySummaryArgs> {
		private static Logger log = Logger.GetLogger(typeof(ActivitySummaryListenerAction).FullName);
		private Action<ActivitySummaryArgs> hand;
		private  string forActivityAnalysis;
		public ActivitySummaryListenerAction( string name, Action<ActivitySummaryArgs> handler ) {
			this.hand = handler;
			this.forActivityAnalysis = name;
		}

		public void OnCompleted() {
			log.info("{0}: Subscription completed", forActivityAnalysis);
		}

		public void OnError( Exception error ) {
			log.severe("{0}: Exception processing: {1}", error, error.Message );
		}

		public void OnNext( ActivitySummaryArgs value ) {
			hand(value);
		}
	}

	/// <summary>
	/// This is a utility class which encapsulates the timing of execution of a section of
	/// code.  The StartNext() and EndNext() methods provide the two functions needed before
	/// and after, respectively, the code block.
	/// </summary>
	public class ActivityAnalysis : IObservable<ActivitySummaryArgs>, ISubscribeHandler<ActivitySummaryArgs> {
		private static Logger log = Logger.GetLogger(typeof(ActivityAnalysis).FullName);
		private ThreadLocal<DateTime> time;
		private double averageWindow = 100;
		private double span;
		private EventProcessor<ActivitySummaryArgs> proc;
		private DateTime lastLog = DateTime.Now;

		public override string ToString() {
			return "ActivityAnalysis: " + Name;
		}
		/// <summary>
		/// 
		/// </summary>
		/// <param name="name"></param>
		public ActivityAnalysis( string name ) {
			Name = name;
			time = new ThreadLocal<DateTime>();
			AverageSpan = new TimeSpan(0, 0, 10);
		}

		public ActivityAnalysis( string name, Action<ActivitySummaryArgs> listener ) {
			Name = name;
			time = new ThreadLocal<DateTime>();
			AverageSpan = new TimeSpan(0, 0, 10);
			Subscribe(new ActivitySummaryListenerAction(name, listener));
		}

		/// <summary>
		/// 
		/// </summary>
		/// <param name="name"></param>
		/// <param name="sub"></param>
		public ActivityAnalysis( string name, IObserver<ActivitySummaryArgs> sub ) {
			Name = name;
			time = new ThreadLocal<DateTime>();
			AverageSpan = new TimeSpan(0, 0, 10);
			Subscribe(sub);
		}

		public T TimeBlock<T>( Func<T> func ) {
			StartNext();
			try {
				return func();
			} finally {
				EndNext();
			}
		}

		public T TimeBlock<A1, T>( A1 arg, Func<A1, T> func ) {
			StartNext();
			try {
				return func(arg);
			} finally {
				EndNext();
			}
		}

		public T TimeBlock<A1, A2, T>( A1 arg1, A2 arg2, Func<A1, A2, T> func ) {
			StartNext();
			try {
				return func(arg1, arg2);
			} finally {
				EndNext();
			}
		}

		public T TimeBlock<A1, A2, A3, T>( A1 arg1, A2 arg2, A3 arg3, Func<A1, A2, A3, T> func ) {
			StartNext();
			try {
				return func(arg1, arg2, arg3);
			} finally {
				EndNext();
			}
		}

		public T TimeBlock<A1, A2, A3, A4, T>( A1 arg1, A2 arg2, A3 arg3, A4 arg4, Func<A1, A2, A3, A4, T> func ) {
			StartNext();
			try {
				return func(arg1, arg2, arg3, arg4);
			} finally {
				EndNext();
			}
		}

		public void TimeAction( Action func ) {
			StartNext();
			try {
				func();
			} finally {
				EndNext();
			}
		}

		public void TimeAction<A1>( A1 arg1, Action<A1> func ) {
			StartNext();
			try {
				func(arg1);
			} finally {
				EndNext();
			}
		}

		public void TimeAction<A1, A2>( A1 arg1, A2 arg2, Action<A1, A2> func ) {
			StartNext();
			try {
				func(arg1, arg2);
			} finally {
				EndNext();
			}
		}

		public void TimeAction<A1, A2, A3>( A1 arg1, A2 arg2, A3 arg3, Action<A1, A2, A3> func ) {
			StartNext();
			try {
				func(arg1, arg2, arg3);
			} finally {
				EndNext();
			}
		}

		public void TimeAction<A1, A2, A3, A4>( A1 arg1, A2 arg2, A3 arg3, A4 arg4, Action<A1, A2, A3, A4> func ) {
			StartNext();
			try {
				func(arg1, arg2, arg3, arg4);
			} finally {
				EndNext();
			}
		}

		/// <summary>
		/// Create an instance 
		/// </summary>
		/// <param name="name"></param>
		/// <param name="span"></param>
		/// <param name="windowSize"></param>
		public ActivityAnalysis( string name, TimeSpan span, int windowSize ) {
			Name = name;
			time = new ThreadLocal<DateTime>();
			AverageSpan = span;
			averageWindow = windowSize;
		}
		public ActivityAnalysis( string name, TimeSpan span, int windowSize, Action<ActivitySummaryArgs> listener ) {
			Name = name;
			time = new ThreadLocal<DateTime>();
			AverageSpan = span;
			averageWindow = windowSize;
			Subscribe(new ActivitySummaryListenerAction(name, listener));
		}
		
		public ActivityAnalysis( string name, TimeSpan span, int windowSize, IObserver<ActivitySummaryArgs> obs ) {
			Name = name;
			time = new ThreadLocal<DateTime>();
			AverageSpan = span;
			averageWindow = windowSize;
			Subscribe(obs);
		}

		/// <summary>
		/// 
		/// </summary>
		public void StartNext() {
			time.Value = DateTime.Now;
		}

		/// <summary>
		/// 
		/// </summary>
		public TimeSpan EndNext() {
			TimeSpan tspan = DateTime.Now - time.Value;
			TimeSpan workspan;
			lock( this ) {
				span = ( ( ( span * ( averageWindow - 1 ) ) + ( tspan.TotalMilliseconds ) ) / averageWindow );
				workspan = DateTime.Now - lastLog;
			}
			if( workspan > AverageSpan ) {
				lastLog = DateTime.Now;
				log.fine("[{0}] averaged({1}) {2:N} millis",
					Name, averageWindow, span);
				if( proc != null ) {
					proc.OnNext(new ActivitySummaryArgs(Name, span, AverageSpan, averageWindow));
				}
			}
			return workspan;
		}
		public string Name { get; set; }

		public TimeSpan AverageSpan { get; set; }

		public IDisposable Subscribe( Action<ActivitySummaryArgs> listener ) {
			return Subscribe(new ActivitySummaryListenerAction(Name, listener));
		}

		/// <summary>
		/// 
		/// </summary>
		/// <param name="observer"></param>
		/// <returns></returns>
		public IDisposable Subscribe( IObserver<ActivitySummaryArgs> observer ) {
			lock( this ) {
				if( proc == null ) {
					proc = new EventProcessor<ActivitySummaryArgs>("EventProcessor-"+Name, this);
				}
			}
			return proc.Subscribe(observer);
		}

		/// <summary>
		/// Interface private method, not public API.
		/// </summary>
		/// <param name="obs"></param>
		public void Subscribed( IObserver<ActivitySummaryArgs> obs ) {
			log.info("{0}: Added subscriber {1}", this, obs);
		}
	}
}
